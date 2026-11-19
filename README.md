# ColdChain_IOT
Système IoT de surveillance de la chaîne du froid : un capteur DHT11 envoie température et humidité à une API Django REST, qui enregistre les mesures et alerte par email et Telegram dès que la température dépasse une degrée précise

# ❄️ ColdChain : surveillance de la chaîne du froid

ColdChain est un système IoT de surveillance de température et d'humidité pour la chaîne du froid (chambres froides, réfrigérateurs de médicaments ou de vaccins, stockage alimentaire).

Un capteur **DHT11** relié à un microcontrôleur (ESP8266 / ESP32 / Arduino) envoie ses mesures à intervalles réguliers à une **API Django REST**. Les données sont enregistrées en base et, si la température dépasse le seuil autorisé, une **alerte est envoyée immédiatement par email et par Telegram**.

---

## ✨ Fonctionnalités

- 📡 Réception des mesures du capteur via une API REST (`POST /post/`)
- 💾 Enregistrement de l'historique des mesures (SQLite)
- 📋 Consultation de toutes les mesures (`GET /list/`)
- 🚨 Alerte automatique quand la température dépasse **28 °C** :
  - 📧 email via SMTP (Gmail)
  - 💬 message Telegram via un bot
- 🛡️ Envois d'alertes indépendants : un échec de l'email n'empêche pas l'envoi Telegram, et la mesure est toujours enregistrée

---

## 🏗️ Architecture

```
┌──────────────┐   HTTP POST (JSON)   ┌──────────────────────┐
│  DHT11 +     │ ───────────────────▶ │  Django + DRF        │
│  ESP/Arduino │   toutes les 5 min   │  (API ColdChain)     │
└──────────────┘                      └──────────┬───────────┘
                                                 │
                         ┌───────────────────────┼───────────────────────┐
                         ▼                       ▼                       ▼
                   ┌──────────┐           ┌────────────┐          ┌────────────┐
                   │  SQLite  │           │ Email SMTP │          │  Telegram  │
                   │ (mesures)│           │  (alerte)  │          │  (alerte)  │
                   └──────────┘           └────────────┘          └────────────┘
```

---

## 🧰 Technologies

| Partie        | Outils                                        |
|---------------|-----------------------------------------------|
| Matériel      | Capteur DHT11, ESP8266 / ESP32 / Arduino      |
| Backend       | Python 3, Django, Django REST Framework       |
| Base de données | SQLite                                      |
| Alertes       | SMTP (Gmail), pyTelegramBotAPI (`telebot`)    |

---

## 📁 Structure du projet

```
ColdChain/
├── manage.py
├── db.sqlite3
├── ColdChain/            # configuration du projet (settings, urls)
└── DHT/                  # application principale
    ├── models.py         # modèle Dht11 (mesures)
    ├── serializers.py    # DHT11serialize
    ├── api.py            # vues API : Dlist, Dhtviews + alertes
    └── urls.py
```

---

## 🚀 Installation

### 1. Cloner le projet et créer l'environnement virtuel

```bash
git clone <url-du-depot> ColdChain
cd ColdChain
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Installer les dépendances

```bash
pip install django djangorestframework pyTelegramBotAPI
```

### 3. Configurer `settings.py`

```python
# Email (Gmail)
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.gmail.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "votre.adresse@gmail.com"
EMAIL_HOST_PASSWORD = "mot-de-passe-d-application"  # mot de passe d'application Gmail

# Telegram
TELEGRAM_TOKEN = "token-fourni-par-BotFather"
TELEGRAM_CHAT_ID = "votre-chat-id"
```

Pour tester sans envoyer de vrais emails, remplacez temporairement le backend :

```python
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

### 4. Créer la base de données

```bash
python manage.py migrate
python manage.py createsuperuser
```

### 5. Lancer le serveur

Pour que le capteur puisse joindre le serveur sur le réseau local, lancez-le sur l'adresse IP de la machine :

```bash
python manage.py runserver <IP-de-la-machine>:8000
```

Pensez à ajouter cette IP dans `ALLOWED_HOSTS` de `settings.py`.

---

## 🔌 API

### `POST /post/` : envoyer une mesure

Corps de la requête (JSON) :

```json
{
  "temperature": 24.5,
  "humidity": 61.0
}
```

Si `temperature > 28`, une alerte est envoyée par email et Telegram.

### `GET /list/` : lister toutes les mesures

Réponse :

```json
{
  "data": [
    { "id": 1, "temperature": 24.5, "humidity": 61.0, "dt": "2026-11-12T20:51:29Z" }
  ]
}
```

> Les champs exacts dépendent du modèle `Dht11` défini dans `DHT/models.py`.

---

## 🔧 Côté microcontrôleur

Le microcontrôleur lit le DHT11 puis envoie un `POST` JSON vers `http://<IP-du-serveur>:8000/post/`. L'intervalle entre deux mesures se règle avec `delay()` :

```cpp
delay(5UL * 60 * 1000); // une mesure toutes les 5 minutes
```

Le suffixe `UL` (unsigned long) évite un dépassement de capacité sur les cartes où `int` fait 16 bits (Arduino Uno / Nano).

---

## 🛠️ Dépannage

| Problème | Solution |
|----------|----------|
| `Error: That port is already in use.` | Un ancien serveur tourne encore : `sudo fuser -k 8000/tcp`, ou utilisez un autre port. |
| `socket.gaierror: [Errno -2] Name or service not known` | `EMAIL_HOST` est incorrect ou vide, ou le réseau bloque le DNS/SMTP. Vérifiez `settings.py` et testez `getent hosts smtp.gmail.com`. |
| Vider les mesures de test | `python manage.py shell -c "from DHT.models import Dht11; Dht11.objects.all().delete()"` |
| Réinitialiser toute la base | `python manage.py flush` (supprime aussi les utilisateurs) |

---

## 🗺️ Évolutions prévues

- 📱 Application mobile (Flutter) avec sonnerie en boucle jusqu'à l'acquittement par le technicien
- ✅ Suivi des alertes : qui a acquitté, et quand
- ⏫ Escalade automatique si une alerte n'est pas acquittée (autre technicien, appel téléphonique)
- 📊 Tableau de bord avec graphiques de température et d'humidité
- ⚙️ Seuils d'alerte configurables par chambre froide

---

## 👤 Auteur

**Ahmed El Mir**
📧 ahmed.elmir.dev@gmail.com 📧 ahmedelmir095@gmail.com
