# 🔍 LAB SonarQube — Intégration CI/CD

> **Mastère CCDAD – M1** | Enseignant : Hedi MAGROUN  
> **Étudiant A — Expert Qualité** : Youssef Marouani  
> **Environnement** : Ubuntu 24.04 (VMware) · Docker · Jenkins · SonarQube Community

---

## 1. Le Besoin

### Problématique métier

Dans un projet logiciel professionnel, sans outil d'analyse automatique du code :

- La **dette technique** s'accumule silencieusement — du code mal structuré devient de plus en plus coûteux à maintenir.
- Des **bugs critiques** passent en production : `NullPointerException`, variables non initialisées, conditions toujours vraies.
- Des **vulnérabilités de sécurité** sont introduites : mots de passe hardcodés dans le code source, secrets exposés, injections potentielles.
- La **couverture de tests** est insuffisante sans mesure objective.

### Solution apportée

**SonarQube** est intégré dans le pipeline Jenkins pour inspecter automatiquement chaque commit. Si le code ne respecte pas le seuil de qualité défini, **le pipeline est bloqué** et aucun artefact n'est publié sur Nexus.

---

## 2. Les Concepts

### Quality Gate

Un Quality Gate est un ensemble de conditions que le code doit respecter pour que l'analyse soit considérée comme réussie. Le Quality Gate par défaut **"Sonar way"** vérifie :

- 0 nouveau bug de sévérité `BLOCKER` ou `CRITICAL`
- 0 nouvelle vulnérabilité de sécurité
- Couverture du nouveau code > 80%
- Duplication du nouveau code < 3%

➡️ Si une condition échoue → `Quality gate: ERROR` → pipeline arrêté.

### Clean as You Code

Méthodologie SonarSource qui consiste à analyser **uniquement le nouveau code** introduit à chaque commit. Cela permet d'améliorer progressivement la qualité sans bloquer le développement sur du code legacy.

### SonarQube Scanner

Agent exécuté dans Jenkins qui :
1. Analyse les fichiers sources
2. Envoie un rapport compressé au serveur SonarQube
3. Attend le verdict du Quality Gate via webhook

### Webhook

Mécanisme par lequel SonarQube notifie Jenkins du résultat de l'analyse via une requête HTTP POST. Sans webhook, Jenkins attendrait indéfiniment (timeout).

```
Jenkins → lance l'analyse → SonarQube
SonarQube → analyse terminée → POST http://jenkins:8080/sonarqube-webhook/
Jenkins → reçoit le verdict → continue ou arrête le pipeline
```

---

## 3. Architecture de la solution

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Network: cicd-network            │
│                                                             │
│   ┌─────────────────┐          ┌─────────────────────────┐  │
│   │    Jenkins      │  Analyse │      SonarQube          │  │
│   │  :8080          │ ───────► │        :9000            │  │
│   │                 │          │                         │  │
│   │                 │ ◄─────── │   Webhook (verdict)     │  │
│   └────────┬────────┘          └─────────────────────────┘  │
│            │                                                │
└────────────┼────────────────────────────────────────────────┘
             │ pull
    ┌─────────▼────────┐
    │   GitHub Repo    │
    │  (Jenkinsfile +  │
    │   code source)   │
    └──────────────────┘

Flux : commit → Jenkins détecte → build → analyse Sonar
       → Quality Gate OK ✅ → Nexus
       → Quality Gate ERROR ❌ → pipeline arrêté
```

---

## 4. Mise en place technique

### 4.1 Prérequis système

SonarQube utilise Elasticsearch en interne qui nécessite une valeur système minimale :

```bash
sudo sysctl -w vm.max_map_count=262144
```

> ⚠️ Sans cette commande, SonarQube refuse de démarrer.

Correction DNS Docker (nécessaire sur certaines VMs) :

```bash
# /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}

sudo systemctl restart docker
```

---

### 4.2 Réseau Docker partagé

```bash
docker network create cicd-network
```

Ce réseau permet à Jenkins et SonarQube de communiquer par **nom de conteneur** (`http://sonarqube:9000`) sans dépendre des adresses IP.

---

### 4.3 Installation SonarQube

```bash
docker run -d \
  --name sonarqube \
  --network cicd-network \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

Vérification du démarrage :

```bash
docker logs -f sonarqube
# Attendre : SonarQube is operational
```

Accès : **http://localhost:9000** · Login : `admin` · Password : `admin` (à modifier)

---

### 4.4 Installation Jenkins

```bash
docker run -d \
  --name jenkins \
  --network cicd-network \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

Récupération du mot de passe initial :

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Accès : **http://localhost:8080**

---

### 4.5 Configuration du plugin SonarQube

Dans Jenkins :

1. **Manage Jenkins → Plugins → Available plugins**
   - Installer `SonarQube Scanner`

2. **Manage Jenkins → Tools → SonarQube Scanner**
   - Name : `SonarScanner`
   - ✅ Install automatically

3. **Manage Jenkins → System → SonarQube servers**
   - ✅ Enable injection of SonarQube server configuration
   - Name : `SonarQube`
   - Server URL : `http://sonarqube:9000`
   - Token : `sonarqube-token` (credential Jenkins)

---

### 4.6 Génération du token

Sur **http://localhost:9000** :

```
Mon avatar → My Account → Security → Generate Token
  Name       : jenkins-token
  Type       : Project Analysis Token
  Project    : pipeline-cicd
  Expiration : No expiration
```

Ajout dans Jenkins :

```
Manage Jenkins → Credentials → System → Global credentials
  Kind        : Secret text
  Secret      : <coller le token>
  ID          : sonarqube-token
```

---

### 4.7 Configuration du Webhook

Sur **http://localhost:9000** :

```
Administration → Configuration → Webhooks → Create
  Name : jenkins
  URL  : http://jenkins:8080/sonarqube-webhook/
```

> 💡 C'est ce webhook qui permet au stage `waitForQualityGate` de fonctionner. Sans lui, Jenkins timeout après 2 minutes.

---

### 4.8 Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        SONAR_SCANNER_HOME = tool 'SonarScanner'
        // tool 'SonarScanner' résout le chemin d'installation
        // défini dans Jenkins Tools
    }

    stages {

        stage('Checkout') {
            steps {
                // Jenkins récupère automatiquement le code
                // depuis le repo Git configuré dans le pipeline
                echo 'Code récupéré depuis Git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // withSonarQubeEnv injecte automatiquement :
                // - SONAR_HOST_URL=http://sonarqube:9000
                // - SONAR_AUTH_TOKEN=<token configuré>
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=pipeline-cicd \
                          -Dsonar.sources=src \
                          -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // Attend le webhook SonarQube (max 2 min)
                // abortPipeline: true = arrête tout si ERROR
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                // Ce stage n'est atteint QUE si le Quality Gate
                // est passé avec succès
                echo 'Artefact publié sur Nexus (Étudiant B)'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline ÉCHOUÉ : Quality Gate non passé !'
        }
        success {
            echo 'Pipeline RÉUSSI : code validé et publié !'
        }
    }
}
```

---

## 5. Tests et Validation

### Scénario 1 — Code propre ✅

Code source sans vulnérabilités → Quality Gate `OK` :

```
SonarQube task completed. Quality gate is 'OK'
Pipeline RÉUSSI : code validé !
Finished: SUCCESS
```

**Résultat :** Le pipeline a traversé tous les stages jusqu'à `Publish to Nexus`.

---

### Scénario 2 — Mauvais code ❌

Après modification de `App.java` avec des vulnérabilités intentionnelles :

```java
public class App {
    public static void main(String[] args) {
        String password = "admin123";        // vulnérabilité : secret hardcodé
        String secret   = "superSecret456"; // vulnérabilité : 2ème secret
        String s = null;
        System.out.println(s.length());      // bug : NullPointerException garanti

        try {
            int x = 1 / 0;
        } catch (Exception e) {
            // bug : exception silencieusement ignorée
        }
    }
}
```

Résultat dans Jenkins :

```
SonarQube task completed. Quality gate is 'ERROR'
ERROR: Pipeline aborted due to quality gate failure: ERROR
Stage "Publish to Nexus" skipped due to earlier failure(s)
Pipeline ÉCHOUÉ : Quality Gate non passé !
Finished: FAILURE
```

**Résultat :** Le pipeline s'est arrêté au Quality Gate. L'artefact n'a pas été publié sur Nexus. ✅

---



## 7. Sources officielles

- 📖 [SonarQube Documentation officielle](https://docs.sonarsource.com/sonarqube/latest/)
- 🔒 [Quality Gates — SonarSource](https://docs.sonarsource.com/sonarqube/latest/user-guide/quality-gates/)
- 🔌 [Jenkins SonarQube Plugin](https://plugins.jenkins.io/sonar/)
- 🐳 [Docker Hub — sonarqube](https://hub.docker.com/_/sonarqube)
- 🐳 [Docker Hub — jenkins](https://hub.docker.com/r/jenkins/jenkins)
- 💻 [Repo GitHub du projet](https://github.com/Youssef-Marouani/pipeline-cicd)
