# TP Jenkins — Pipeline CI/CD avec Docker, ECR et déploiement sur EC2

## Objectif

Mettre en place une pipeline CI/CD complète avec Jenkins :

- Un **push sur GitHub/GitLab** déclenche automatiquement la pipeline
- Jenkins **build une image Docker** d'une page web Nginx
- L'image est **poussée sur Amazon ECR** (ou Docker Hub)
- Jenkins se connecte en SSH à un **EC2 applicatif** pour mettre à jour le conteneur

## Architecture cible

```
Developer (push)
      │
      ▼
GitHub / GitLab  ──(webhook)──►  Jenkins EC2 (port 8080)
                                        │
                          ┌─────────────┴──────────────┐
                          ▼                            ▼
                   Build image Docker           Push image vers
                   & tag avec commit SHA        Amazon ECR / Docker Hub
                                                       │
                                                       ▼
                                              SSH vers App EC2
                                              └─ docker pull
                                              └─ docker stop / rm
                                              └─ docker run (nginx)
```

**Deux EC2 sont nécessaires :**

| Machine        | Rôle                              | Ports ouverts        |
|----------------|-----------------------------------|----------------------|
| `EC2-Jenkins`  | Héberge Jenkins (via Docker)      | 22, 8080             |
| `EC2-App`      | Héberge l'application Nginx       | 22, 80               |

---

## Partie 1 — Préparation du dépôt Git

> La création du dépôt et la liaison avec le remote sont supposées connues. Voici uniquement les fichiers à placer dans le projet.

### Structure du dépôt

```
mon-projet/
├── index.html
├── Dockerfile
└── Jenkinsfile
```

### `index.html`

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <title>Mon App Jenkins</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      margin: 0;
      background: #1a1a2e;
      color: #e0e0e0;
    }
    .card {
      text-align: center;
      background: #16213e;
      padding: 3rem;
      border-radius: 12px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.4);
    }
    h1 { color: #0f3460; font-size: 2.5rem; }
    p  { color: #a0a0b0; }
    .badge {
      display: inline-block;
      background: #e94560;
      color: white;
      padding: 0.4rem 1rem;
      border-radius: 20px;
      margin-top: 1rem;
      font-weight: bold;
    }
  </style>
</head>
<body>
  <div class="card">
    <h1>🚀 Déployé avec Jenkins</h1>
    <p>Pipeline CI/CD opérationnelle</p>
    <span class="badge">Version 1.0</span>
  </div>
</body>
</html>
```

> **À chaque modification de cette page et push, la pipeline se déclenchera et déploiera automatiquement la nouvelle version.**

### `Dockerfile`

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

### `Jenkinsfile`

> Deux variantes proposées : **ECR** (AWS) ou **Docker Hub**. Choisissez celle adaptée à votre environnement.

#### Variante A — Amazon ECR

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'                         // Adaptez à votre région
        ECR_REGISTRY    = '123456789012.dkr.ecr.us-east-1.amazonaws.com'  // Votre ID de compte AWS
        ECR_REPO        = 'mon-app-nginx'
        IMAGE_TAG       = "${env.GIT_COMMIT[0..6]}"
        APP_EC2_HOST    = credentials('app-ec2-host')         // IP de l'EC2 applicatif
        APP_EC2_USER    = 'ubuntu'
        CONTAINER_NAME  = 'app-nginx'
        APP_PORT        = '80'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        stage('Build Image') {
            steps {
                script {
                    sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:latest"
                }
            }
        }

        stage('Push vers ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
                    sh "docker push ${ECR_REGISTRY}/${ECR_REPO}:latest"
                }
            }
        }

        stage('Déployer sur App EC2') {
            steps {
                sshagent(credentials: ['app-ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} '
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker pull ${ECR_REGISTRY}/${ECR_REPO}:latest
                            docker stop ${CONTAINER_NAME} || true
                            docker rm   ${CONTAINER_NAME} || true
                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                -p ${APP_PORT}:80 \
                                --restart unless-stopped \
                                ${ECR_REGISTRY}/${ECR_REPO}:latest
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Déploiement réussi — image : ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ La pipeline a échoué. Consultez les logs ci-dessus."
        }
        always {
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REGISTRY}/${ECR_REPO}:latest       || true"
        }
    }
}
```

#### Variante B — Docker Hub

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USER  = 'votre-username'
        IMAGE_NAME      = "${DOCKERHUB_USER}/mon-app-nginx"
        IMAGE_TAG       = "${env.GIT_COMMIT[0..6]}"
        APP_EC2_HOST    = credentials('app-ec2-host')
        APP_EC2_USER    = 'ubuntu'
        CONTAINER_NAME  = 'app-nginx'
        APP_PORT        = '80'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Commit : ${env.GIT_COMMIT}"
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('Push vers Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh "echo $DH_PASS | docker login -u $DH_USER --password-stdin"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Déployer sur App EC2') {
            steps {
                sshagent(credentials: ['app-ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} '
                            docker pull ${IMAGE_NAME}:latest
                            docker stop ${CONTAINER_NAME} || true
                            docker rm   ${CONTAINER_NAME} || true
                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                -p ${APP_PORT}:80 \
                                --restart unless-stopped \
                                ${IMAGE_NAME}:latest
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Déploiement réussi — image : ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ La pipeline a échoué."
        }
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${IMAGE_NAME}:latest       || true"
        }
    }
}
```

---

## Partie 2 — EC2 Jenkins : installation et configuration

### 2.1 Lancer l'instance EC2 Jenkins

Sur la console AWS, créez une instance EC2 :

- **AMI :** Ubuntu Server 24.04 LTS
- **Type :** t2.medium (minimum recommandé pour Jenkins)
- **Stockage :** 20 Go minimum
- **Security Group :** ouvrez les ports **22** (SSH) et **8080** (Jenkins)

### 2.2 Connexion et installation de Docker

Connectez-vous en SSH à l'instance Jenkins :

```bash
ssh -i votre-cle.pem ubuntu@<IP_JENKINS_EC2>
```

Mettez à jour le système et installez Docker :

```bash
sudo apt-get update -y && sudo apt-get upgrade -y

# Installer Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Ajouter l'utilisateur ubuntu au groupe docker
sudo usermod -aG docker ubuntu

# Activer Docker au démarrage
sudo systemctl enable docker
sudo systemctl start docker

# Vérifier l'installation (reconnectez-vous pour que le groupe soit pris en compte)
docker --version
```

> ⚠️ **Reconnectez-vous en SSH** après `usermod` pour que les droits Docker soient actifs.

### 2.3 Installer Jenkins via Docker

```bash
# Créer un volume persistant pour les données Jenkins
docker volume create jenkins_data

# Lancer Jenkins
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

> Le montage de `/var/run/docker.sock` permet à Jenkins d'utiliser Docker de l'hôte depuis l'intérieur du conteneur.

### 2.4 Donner à Jenkins l'accès à Docker

Le conteneur Jenkins tourne avec l'utilisateur `jenkins` qui n'a pas accès à la socket Docker par défaut.

```bash
# Trouver le GID du groupe docker sur l'hôte
getent group docker
# Exemple : docker:x:998:ubuntu  → GID = 998

# Donner les droits à l'utilisateur jenkins dans le conteneur
docker exec -u root jenkins groupadd -g 998 docker_host
docker exec -u root jenkins usermod -aG docker_host jenkins

# Redémarrer le conteneur
docker restart jenkins
```

> **Note :** Le GID peut varier selon l'installation. Adaptez la valeur `998` au résultat de votre commande `getent group docker`.

### 2.5 Accéder à Jenkins et terminer l'installation

Dans votre navigateur, accédez à :

```
http://<IP_JENKINS_EC2>:8080
```

Récupérez le mot de passe initial :

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copiez-le dans l'interface, puis :

1. Cliquez sur **"Install suggested plugins"**
2. Créez votre compte administrateur
3. Validez l'URL Jenkins (laissez la valeur par défaut)

### 2.6 Installer les plugins nécessaires

Allez dans **Manage Jenkins → Plugins → Available plugins** et installez :

- **SSH Agent** — pour se connecter à l'EC2 applicatif via SSH
- **Docker Pipeline** — pour les commandes Docker dans le Jenkinsfile
- **Git** — normalement déjà installé
- **Pipeline** — normalement déjà installé

Redémarrez Jenkins après l'installation.

### 2.7 Installer AWS CLI dans le conteneur Jenkins (si vous utilisez ECR)

```bash
docker exec -u root jenkins bash -c "
  apt-get update -y && \
  apt-get install -y curl unzip && \
  curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o awscliv2.zip && \
  unzip awscliv2.zip && \
  ./aws/install && \
  rm -rf awscliv2.zip aws
"
docker restart jenkins
```

Vérifiez :

```bash
docker exec jenkins aws --version
```

---

## Partie 3 — EC2 Applicatif : préparation

### 3.1 Lancer l'instance EC2 App

Sur la console AWS, créez une seconde instance EC2 :

- **AMI :** Ubuntu Server 24.04 LTS
- **Type :** t2.micro suffit
- **Stockage :** 10 Go
- **Security Group :** ouvrez les ports **22** (SSH) et **80** (HTTP)

### 3.2 Installation de Docker sur l'EC2 App

```bash
ssh -i votre-cle.pem ubuntu@<IP_APP_EC2>

sudo apt-get update -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
sudo systemctl enable docker
sudo systemctl start docker
```

Reconnectez-vous en SSH, puis vérifiez :

```bash
docker --version
```

### 3.3 Configurer l'accès AWS ECR sur l'EC2 App (si vous utilisez ECR)

L'EC2 App doit pouvoir tirer les images depuis ECR. Deux méthodes :

**Méthode recommandée : IAM Role**

1. Dans la console AWS, créez un **IAM Role** avec la policy `AmazonEC2ContainerRegistryReadOnly`
2. Attachez ce rôle à votre **EC2 App** (Actions → Security → Modify IAM role)

**Méthode alternative : AWS CLI**

```bash
sudo apt-get install -y awscli
aws configure  # Entrez vos Access Key, Secret Key et région
```

---

## Partie 4 — Configuration des credentials Jenkins

Tout secret (clé SSH, mot de passe, token) doit être stocké dans Jenkins sous forme de **credential**, jamais en clair dans le Jenkinsfile.

Allez dans **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**.

### 4.1 Clé SSH pour l'EC2 App

> **Point important :** lors de la création de l'EC2 App sur AWS, vous avez choisi une paire de clés `.pem`. C'est **votre** clé personnelle pour vous y connecter. Il ne faut pas la donner à Jenkins.
>
> La bonne pratique est de générer une **paire de clés dédiée à Jenkins**, puis d'ajouter sa clé publique sur l'EC2 App. Vous aurez ainsi deux accès SSH distincts et révocables indépendamment.

**Étape 1 — Générer une paire de clés SSH dédiée, dans le conteneur Jenkins**

```bash
docker exec -u jenkins jenkins ssh-keygen -t ed25519 -C "jenkins-deploy" \
  -f /var/jenkins_home/.ssh/deploy_key -N ""
```

Affichez la clé publique générée :

```bash
docker exec jenkins cat /var/jenkins_home/.ssh/deploy_key.pub
```

Copiez le contenu affiché (une seule ligne commençant par `ssh-ed25519 ...`).

**Étape 2 — Ajouter la clé publique Jenkins sur l'EC2 App**

Connectez-vous à l'EC2 App **avec votre clé .pem AWS habituelle** :

```bash
ssh -i votre-cle.pem ubuntu@<IP_APP_EC2>
```

Puis ajoutez la clé publique de Jenkins :

```bash
echo "COLLEZ_ICI_LA_CLE_PUBLIQUE_JENKINS" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

L'EC2 App accepte maintenant deux connexions SSH : la vôtre (via `.pem`) et celle de Jenkins (via sa paire dédiée).

**Étape 3 — Ajouter la clé privée Jenkins dans les credentials Jenkins**

Affichez la clé privée :

```bash
docker exec jenkins cat /var/jenkins_home/.ssh/deploy_key
```

Dans **Add Credentials** :

| Champ        | Valeur                                                                      |
|--------------|-----------------------------------------------------------------------------|
| Kind         | SSH Username with private key                                               |
| ID           | `app-ec2-ssh-key`                                                           |
| Username     | `ubuntu`                                                                    |
| Private Key  | Enter directly → collez le contenu complet de la clé privée (de `-----BEGIN` à `-----END`) |

### 4.2 IP de l'EC2 App (Secret Text)

| Champ  | Valeur                         |
|--------|--------------------------------|
| Kind   | Secret text                    |
| ID     | `app-ec2-host`                 |
| Secret | `<IP_PUBLIQUE_DE_APP_EC2>`     |

### 4.3 Credentials Docker Hub (si variante B)

| Champ    | Valeur                        |
|----------|-------------------------------|
| Kind     | Username with password        |
| ID       | `dockerhub-credentials`       |
| Username | Votre username Docker Hub     |
| Password | Votre mot de passe Docker Hub |

### 4.4 Accès Jenkins à Amazon ECR (si variante A)

Pour que Jenkins puisse pousser des images vers ECR, il doit s'authentifier auprès d'AWS. Deux méthodes :

**Méthode recommandée — IAM Role sur l'EC2 Jenkins**

C'est l'approche la plus propre : aucune clé à stocker, AWS gère l'authentification automatiquement.

1. Dans la console AWS, créez un **IAM Role** de type *EC2* avec la policy `AmazonEC2ContainerRegistryFullAccess`
2. Attachez ce rôle à votre **EC2 Jenkins** : console EC2 → sélectionnez l'instance → **Actions → Security → Modify IAM role**
3. C'est tout. La commande `aws ecr get-login-password` du Jenkinsfile fonctionnera sans configuration supplémentaire.

**Méthode alternative — Access Key stockée dans Jenkins**

Si vous ne pouvez pas utiliser de IAM Role, stockez les credentials AWS dans Jenkins.

Dans **Manage Jenkins → Credentials → Add Credentials**, créez deux **Secret text** :

| ID                      | Secret                  |
|-------------------------|-------------------------|
| `aws-access-key-id`     | Votre AWS Access Key ID |
| `aws-secret-access-key` | Votre AWS Secret Key    |

Puis modifiez le stage ECR dans votre Jenkinsfile :

```groovy
stage('Push vers ECR') {
    steps {
        withCredentials([
            string(credentialsId: 'aws-access-key-id',     variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
            sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REGISTRY}
                docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
            """
        }
    }
}
```

---

## Partie 5 — Créer le job Jenkins

### 5.1 Nouveau job Pipeline

1. Sur le tableau de bord Jenkins, cliquez sur **New Item**
2. Nommez le job (ex. : `deploy-nginx-app`)
3. Sélectionnez **Pipeline** et cliquez **OK**

### 5.2 Configuration du job

**Section "Build Triggers"**

Cochez **"GitHub hook trigger for GITScm polling"** (ou GitLab équivalent).

**Section "Pipeline"**

| Champ               | Valeur                                          |
|---------------------|-------------------------------------------------|
| Definition          | Pipeline script from SCM                        |
| SCM                 | Git                                             |
| Repository URL      | URL HTTPS ou SSH de votre dépôt                 |
| Credentials         | Ajoutez vos credentials Git si dépôt privé      |
| Branch Specifier    | `*/main` (ou `*/master`)                        |
| Script Path         | `Jenkinsfile`                                   |

Cliquez **Save**.

---

## Partie 6 — Configurer le Webhook

Le webhook est le mécanisme qui permet à GitHub/GitLab de notifier Jenkins à chaque push.

### 6.1 GitHub

1. Dans votre dépôt GitHub : **Settings → Webhooks → Add webhook**
2. Renseignez les champs :

| Champ          | Valeur                                              |
|----------------|-----------------------------------------------------|
| Payload URL    | `http://<IP_JENKINS_EC2>:8080/github-webhook/`      |
| Content type   | `application/json`                                  |
| Which events   | **Just the push event**                             |
| Active         | ✅ Coché                                            |

3. Cliquez **Add webhook**. GitHub enverra un ping initial ; vérifiez qu'il retourne un code **200**.

> ⚠️ L'URL doit être **publiquement accessible**. Si vous testez en local, utilisez [ngrok](https://ngrok.com) pour exposer votre Jenkins.

### 6.2 GitLab

1. Dans votre projet GitLab : **Settings → Webhooks**
2. Renseignez :

| Champ          | Valeur                                              |
|----------------|-----------------------------------------------------|
| URL            | `http://<IP_JENKINS_EC2>:8080/gitlab/build_now`     |
| Trigger        | **Push events** ✅                                  |
| SSL            | Décochez si pas de HTTPS                            |

3. Cliquez **Add webhook**, puis **Test → Push events** pour vérifier.

---

## Partie 7 — Créer le dépôt ECR (si variante A)

Si vous utilisez Amazon ECR, créez d'abord votre dépôt :

```bash
# Depuis votre machine locale ou l'EC2 Jenkins
aws ecr create-repository \
  --repository-name mon-app-nginx \
  --region us-east-1
```

Notez l'URI retourné (ex. : `123456789012.dkr.ecr.us-east-1.amazonaws.com/mon-app-nginx`) et mettez-le à jour dans votre `Jenkinsfile`.

---

## Partie 8 — Test de la pipeline

### 8.1 Premier déclenchement manuel

1. Dans Jenkins, ouvrez votre job `deploy-nginx-app`
2. Cliquez **Build Now**
3. Cliquez sur le build `#1` → **Console Output** pour suivre l'exécution

### 8.2 Test du webhook

Modifiez `index.html` (par exemple, changez "Version 1.0" en "Version 2.0"), commitez et poussez :

```bash
git add index.html
git commit -m "Mise à jour version 2.0"
git push origin main
```

Retournez sur Jenkins : un nouveau build doit se déclencher automatiquement dans les secondes qui suivent.

### 8.3 Vérifier l'application

Dans votre navigateur :

```
http://<IP_APP_EC2>
```

Vous devez voir votre page HTML mise à jour. Chaque push déclenchera une mise à jour automatique.

---

## Récapitulatif des credentials Jenkins

| ID                      | Type                        | Usage                              |
|-------------------------|-----------------------------|------------------------------------|
| `app-ec2-ssh-key`       | SSH Username with private key | Connexion SSH à l'EC2 App        |
| `app-ec2-host`          | Secret text                 | IP de l'EC2 App                    |
| `dockerhub-credentials` | Username with password      | Authentification Docker Hub (var B)|

---

## Synthèse de l'architecture mise en place

```
┌─────────────────────────────────────────────────────────────────┐
│                        Flux complet                             │
│                                                                 │
│  1. git push origin main                                        │
│         │                                                       │
│         ▼                                                       │
│  2. GitHub/GitLab envoie un webhook POST → Jenkins:8080         │
│         │                                                       │
│         ▼                                                       │
│  3. Jenkins (EC2-Jenkins) exécute le Jenkinsfile :              │
│     ├─ Checkout du code                                         │
│     ├─ docker build → image taguée avec le SHA du commit        │
│     ├─ docker push → ECR ou Docker Hub                          │
│     └─ SSH vers EC2-App :                                       │
│           ├─ docker pull (nouvelle image)                       │
│           ├─ docker stop / rm (ancien conteneur)                │
│           └─ docker run (nouveau conteneur sur port 80)         │
│                                                                 │
│  4. L'application est accessible sur http://<IP_APP_EC2>        │
└─────────────────────────────────────────────────────────────────┘
```
