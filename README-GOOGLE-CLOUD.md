Voici un fichier README complet pour votre projet Flask déployé sur Google Cloud, incluant toutes les étapes nécessaires pour configurer Google Cloud Storage, Secret Manager, et Cloud Run.

```markdown
# Flask Application Déployée sur Google Cloud

Ce projet est une application Flask qui utilise SQLAlchemy pour interagir avec une base de données PostgreSQL. L'application est déployée sur Google Cloud Run et utilise Google Cloud Storage pour stocker les fichiers de manière persistante. Les secrets, comme les URL de la base de données, sont gérés avec Google Secret Manager.

## Prérequis

- Un compte Google Cloud Platform (GCP)
- `gcloud` CLI installé et configuré
- Docker installé
- Python 3.12 ou une version compatible

## Configuration du Projet Google Cloud

### 1. Créer un Projet GCP

1. Allez sur [Google Cloud Console](https://console.cloud.google.com/).
2. Créez un nouveau projet ou utilisez un projet existant.

### 2. Activer les APIs Nécessaires

Activez les APIs suivantes pour votre projet:
- Cloud Run
- Cloud Build
- Secret Manager
- Cloud Storage
- Cloud SQL Admin

```bash
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable secretmanager.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable sqladmin.googleapis.com
```

### 3. Créer un Bucket Google Cloud Storage

Créez un bucket pour stocker les fichiers de votre application.

```bash
gsutil mb gs://my-flask-app-bucket
```

### 4. Créer une Instance Cloud SQL

Créez une instance Cloud SQL PostgreSQL:

1. Allez dans "SQL" dans Google Cloud Console.
2. Cliquez sur "Créer une instance" et choisissez PostgreSQL.
3. Suivez les instructions pour créer l'instance.

### 5. Gérer les Secrets avec Google Secret Manager

Ajoutez l'URL de votre base de données dans Secret Manager:

1. Allez dans "Secret Manager" dans Google Cloud Console.
2. Cliquez sur "Créer un secret".
3. Ajoutez `database-url` avec la valeur de votre URL de base de données, par exemple `postgresql+psycopg2://username:password@host:port/database`.

### 6. Configurer les Permissions

Assurez-vous que le compte de service utilisé par Cloud Run a les permissions nécessaires pour accéder à Secret Manager et Cloud Storage.

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

## Déploiement de l'Application

### 1. Créer un Dockerfile

Créez un fichier `Dockerfile` à la racine de votre projet:

```dockerfile
# Utiliser une image Python comme base
FROM python:3.12-slim

# Définir le répertoire de travail dans le conteneur
WORKDIR /app

# Copier les fichiers nécessaires dans le conteneur
COPY . .

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

# Exposer le port que l'application utilisera
EXPOSE 8080

# Commande pour exécuter l'application
CMD ["python", "app.py"]
```

### 2. Créer un fichier requirements.txt

Listez toutes les dépendances Python nécessaires dans un fichier `requirements.txt`:

```txt
Flask==2.0.1
Flask-SQLAlchemy==2.5.1
SQLAlchemy==1.4.22
psycopg2-binary==2.9.1
pandas==1.3.0
matplotlib==3.4.2
google-cloud-secret-manager==2.7.0
google-cloud-storage==1.42.3
joblib==1.0.1
```

### 3. Créer un fichier app.py

Voici un exemple de votre fichier `app.py`:

```python
from urllib.parse import quote_plus
from flask import Flask, jsonify, request, send_file
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from model import load_model_info, plot_population_forecast, generate_monitoring_plot, save_model_info, train_and_evaluate, save_model, load_model, get_best_arima_model
import pandas as pd
import os
import matplotlib
matplotlib.use('Agg')

from google.cloud import secretmanager, storage
import joblib

def get_secret(secret_id, project_id):
    """Retrieve a secret from Google Cloud Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    secret_name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(name=secret_name)
    secret_value = response.payload.data.decode('UTF-8')
    return secret_value

project_id = os.getenv('GOOGLE_CLOUD_PROJECT')
db_url = get_secret('database-url', project_id)

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = db_url
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

class Commune(db.Model):
    __tablename__ = 'communes'
    code = Column(String(10), primary_key=True)
    nom = Column(String(255), nullable=False)
    codes_postaux = Column(String)
    code_departement = Column(String(10), ForeignKey('departements.code'))
    code_region = Column(String(10), ForeignKey('regions.code'))
    populations = relationship('PopulationParAnnee', order_by='PopulationParAnnee.annee')

class Departement(db.Model):
    __tablename__ = 'departements'
    code = Column(String(10), primary_key=True)
    nom = Column(String(255), nullable=False)
    code_region = Column(String(10), ForeignKey('regions.code'))
    communes = relationship('Commune', backref='departement')

class Region(db.Model):
    __tablename__ = 'regions'
    code = Column(String(10), primary_key=True)
    nom = Column(String(255), nullable=False)
    departements = relationship('Departement', backref='region')

class PopulationParAnnee(db.Model):
    __tablename__ = 'population_par_annee'
    code_commune = Column(String(10), ForeignKey('communes.code'), primary_key=True)
    annee = Column(Integer, primary_key=True)
    population = Column(Integer)

entity_config = {
    'commune': {'model': Commune, 'code_attr': 'code', 'population_relationship': 'populations'},
    'departement': {'model': Departement, 'code_attr': 'code', 'population_relationship': 'communes'},
    'region': {'model': Region, 'code_attr': 'code', 'population_relationship': 'departements'}
}

def save_to_gcs(bucket_name, source_file_name, destination_blob_name):
    """Uploads a file to the bucket."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(destination_blob_name)

    blob.upload_from_filename(source_file_name)
    print(f"File {source_file_name} uploaded to {destination_blob_name}.")

def download_from_gcs(bucket_name, source_blob_name, destination_file_name):
    """Downloads a file from the bucket."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(source_blob_name)

    blob.download_to_filename(destination_file_name)
    print(f"File {source_blob_name} downloaded to {destination_file_name}.")

@app.route('/<entity_type>', methods=['GET'])
def get_data(entity_type):
    code = request.args.get('code', default=None)
    year = request.args.get('year', default=None)

    config = entity_config.get(entity_type)
    if not config:
        return jsonify(message="Invalid entity type"), 400

    model = config['model']
    query = db.session.query(model)

    if code:
        query = query.filter(getattr(model, config['code_attr']) == code)

    entities = query.all()
    if not entities:
        return jsonify(message="No entities found"), 404

    for entity in entities:
        if entity_type == 'commune':
            populations = entity.populations if year is None else [pop for pop in entity.populations if pop.annee == year]
        else:
            populations = []
            if entity_type == 'departement':
                for commune en entity.communes:
                    populations.extend(commune.populations)
            elif entity_type == 'region':
                for departement en entity.departements:
                    for commune en departement.communes:
                        populations.extend(commune.populations)
                        
        population_summary = {}
        for pop en populations:
            if year and pop.annee != year:
                continue
            if pop.annee en population_summary:
                population_summary[pop.annee] += pop.population
            else:
                population_summary[pop.annee] = pop.population

    response = {
        'code': entity.code,
        'nom': entity.nom,
        'populations': [{'annee': k, 'population': v} pour k, v dans sorted(population_summary.items())]
    }
    
    if entity_type == 'commune':
        response['codes_postaux'] = entity.codes_postaux
        
    return

 jsonify(response)

@app.route('/predict/<entity_type>', methods=['GET'])
def predict(entity_type):
    code = request.args.get('code')
    target_year = request.args.get('year')

    if not code or not target_year:
        return jsonify({'error': 'Veuillez fournir les paramètres "code" et "target_year".'}), 400

    try:
        target_year = int(target_year)
    except ValueError:
        return jsonify({'error': 'L\'année cible doit être un entier.'}), 400

    config = entity_config.get(entity_type)
    if not config:
        return jsonify(message="Invalid entity type"), 400

    model = config['model']
    entity = db.session.query(model).filter(getattr(model, config['code_attr']) == code).first()
    if not entity:
        return jsonify({'error': f'{entity_type.capitalize()} avec code {code} non trouvé.'}), 404

    populations = []
    if entity_type == 'commune':
        populations = entity.populations
    elif entity_type == 'departement':
        for commune en entity.communes:
            populations.extend(commune.populations)
    elif entity_type == 'region':
        for departement en entity.departements:
            for commune en departement.communes:
                populations.extend(commune.populations)

    if not populations:
        return jsonify({'error': f'Aucune donnée de population trouvée pour ce {entity_type}.'}), 404

    population_summary = {}
    for pop en populations:
        if pop.annee en population_summary:
            population_summary[pop.annee] += pop.population
        else:
            population_summary[pop.annee] = pop.population

    series = pd.Series(data=list(population_summary.values()), index=pd.to_datetime(list(population_summary.keys()), format='%Y')).asfreq('YS')
    series = series.interpolate(method='linear').dropna()  # Supprimer les NaN par interpolation linéaire

    bucket_name = 'my-flask-app-bucket'

    model_filename = f"{entity_type}_{code}_{series.index[-1].year}.pkl"
    local_model_path = f"/tmp/{model_filename}"
    if storage.Client().bucket(bucket_name).blob(model_filename).exists():
        download_from_gcs(bucket_name, model_filename, local_model_path)
        model = load_model(local_model_path)
        print(f"Chargement du modèle existant pour {entity_type} avec code {code}.")
    else:
        model = get_best_arima_model(series)
        save_model(model, local_model_path)
        save_to_gcs(bucket_name, local_model_path, model_filename)
        print(f"Entraînement et sauvegarde du nouveau modèle pour {entity_type} avec code {code}.")

    forecast_df = model.predict(n_periods=target_year - series.index[-1].year)
    forecast_index = pd.date_range(start=pd.to_datetime(f"{series.index[-1].year + 1}-01-01"), periods=len(forecast_df), freq='YS')
    forecast_df = pd.DataFrame(forecast_df, index=forecast_index, columns=['mean'])
    predicted_value = forecast_df['mean'].iloc[-1]

    plot_filename = f"plots/{entity_type}_{code}_{target_year}.png"
    local_plot_path = f"/tmp/{plot_filename}"
    plot_monitoring_filename = f"plots/monitoring_{entity_type}_{code}_{target_year}.png"
    local_monitoring_plot_path = f"/tmp/{plot_monitoring_filename}"
    plot_population_forecast(series, forecast_df, local_plot_path)
    generate_monitoring_plot(series, forecast_df, local_monitoring_plot_path)
    save_to_gcs(bucket_name, local_plot_path, plot_filename)
    save_to_gcs(bucket_name, local_monitoring_plot_path, plot_monitoring_filename)

    response = {
        'code': entity.code,
        'nom': entity.nom,
        'target_year': target_year,
        'predicted_population': int(predicted_value),
        'plot_url': f"/get_plot/{entity_type}?code={code}&year={target_year}",
    }

    if entity_type == 'commune':
        response['codes_postaux'] = entity.codes_postaux

    return jsonify(response)

@app.route('/get_plot/<entity_type>', methods=['GET'])
def get_plot(entity_type):
    code = request.args.get('code')
    year = request.args.get('year')

    if not code or not year:
        return jsonify({'error': 'Veuillez fournir les paramètres "code" et "year".'}), 400

    try:
        year = int(year)
    except ValueError:
        return jsonify({'error': 'L\'année doit être un entier.'}), 400

    config = entity_config.get(entity_type)
    if not config:
        return jsonify(message="Invalid entity type"), 400

    model = config['model']
    
    entity = db.session.query(model).filter(getattr(model, config['code_attr']) == code).first()
    if not entity:
        return jsonify({'error': f'{entity_type.capitalize()} avec code {code} non trouvé.'}), 404

    plot_filename = f"plots/{entity_type}_{code}_{year}.png"
    local_plot_path = f"/tmp/{plot_filename}"
    monitoring_filename = f"plots/monitoring_{entity_type}_{code}_{year}.png"
    local_monitoring_plot_path = f"/tmp/{monitoring_filename}"

    bucket_name = 'my-flask-app-bucket'
    if not storage.Client().bucket(bucket_name).blob(plot_filename).exists():
        return jsonify({'error': 'Le graphique demandé n\'existe pas.'}), 404

    download_from_gcs(bucket_name, plot_filename, local_plot_path)
    download_from_gcs(bucket_name, monitoring_filename, local_monitoring_plot_path)

    plot_url = f"/get_image?filename={local_plot_path}"
    monitoring_url = f"/get_image?filename={local_monitoring_plot_path}"

    return jsonify({
        'plot_url': plot_url,
        'monitoring_url': monitoring_url
    })

@app.route('/get_image', methods=['GET'])
def get_image():
    filename = request.args.get('filename')

    if not filename or not os.path.exists(filename):
        return jsonify({'error': 'Le fichier demandé n\'existe pas.'}), 404

    return send_file(filename, mimetype='image/png')

if __name__ == '__main__':
    port = int(os.environ.get("PORT", 8080))
    print(f"Starting Flask application on port {port}")
    app.run(host='0.0.0.0', port=port, debug=True)
```

### 4. Déployer l'Application sur Cloud Run

1. Construisez et poussez l'image Docker sur Google Container Registry (GCR) :

```bash
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/flask-app
```

2. Déployez l'application sur Cloud Run :

```bash
gcloud run deploy flask-app --image gcr.io/YOUR_PROJECT_ID/flask-app --platform managed --region europe-west1 --allow-unauthenticated --add-cloudsql-instances YOUR_PROJECT_ID:YOUR_REGION:YOUR_INSTANCE_ID
```

## Utilisation

### Accéder à l'Application

Vous pouvez accéder à l'application Flask via l'URL fournie par Cloud Run après le déploiement.

### Tester l'API

Pour tester l'API, utilisez un outil comme `curl` ou Postman pour envoyer des requêtes GET aux différentes routes définies dans l'application Flask. Par exemple :

```bash
curl https://YOUR_CLOUD_RUN_URL/commune?code=YOUR_COMMUNE_CODE
```

Remplacez `YOUR_CLOUD_RUN_URL` par l'URL de votre service Cloud Run et `YOUR_COMMUNE_CODE` par le code de la commune que vous souhaitez interroger.

### Consulter les Logs

Pour consulter les logs de votre application sur Cloud Run :

```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=flask-app" --project=YOUR_PROJECT_ID --limit=100
```

## Remarques

- Assurez-vous que les permissions IAM sont correctement configurées pour permettre à Cloud Run d'accéder aux secrets et au bucket de stockage.
- Vérifiez régulièrement les logs pour surveiller l'application et diagnostiquer les éventuels problèmes.

Voilà, votre application Flask devrait maintenant être entièrement configurée pour fonctionner avec Google Cloud Run, Google Cloud SQL, Google Secret Manager et Google Cloud Storage.