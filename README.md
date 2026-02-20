# QR Finder

QR Finder è un'applicazione web completa per il tracciamento di oggetti personali tramite QR code. Gli utenti possono registrarsi, generare QR code univoci per i loro oggetti (chiavi, smartphone, biciclette, ecc.), acquistare etichette fisiche e ricevere notifiche quando i loro oggetti vengono trovati.

## Caratteristiche

- **Registrazione e Login**: Sistema di autenticazione completo con verifica email
- **Generazione QR Code**: QR code univoci per ogni oggetto con short URL
- **Pagamenti Stripe**: Integrazione completa per l'acquisto di etichette
- **Geolocalizzazione**: Tracciamento della posizione quando il QR viene scannerizzato
- **Notifiche**: Email e notifiche sul sito quando gli oggetti vengono trovati
- **Mappa Interattiva**: Visualizzazione della posizione degli oggetti con Leaflet + OpenStreetMap
- **Dashboard Utente**: Gestione completa degli oggetti e delle notifiche

## Requisiti

- PHP >= 8.1
- MariaDB >= 10.4 o MySQL >= 5.7
- Composer
- Account Stripe (per i pagamenti)
- Server SMTP (per le email)

## Installazione

### 1. Clona il repository

```bash
cd /var/www/html
git clone https://github.com/tuouser/qr-finder.git
cd qr-finder
```

### 2. Installa le dipendenze

```bash
composer install
```

### 3. Configura il database

Crea un database MariaDB/MySQL e importa lo schema:

```bash
mysql -u root -p
CREATE DATABASE qrfinder CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
exit
mysql -u root -p qrfinder < database/schema.sql
```

### 4. Configura l'ambiente

Copia il file di esempio e modifica le configurazioni:

```bash
cp .env.example .env
nano .env
```

Modifica le seguenti variabili:

```env
# Database
DB_HOST=localhost
DB_NAME=qrfinder
DB_USER=root
DB_PASS=la_tua_password

# Application
APP_URL=https://qr-finder.com

# JWT (genera una chiave casuale)
JWT_SECRET=chiave_segreta_casuale_di_almeno_32_caratteri

# Stripe
STRIPE_PUBLISHABLE_KEY=pk_test_tua_chiave
STRIPE_SECRET_KEY=sk_test_tua_chiave
STRIPE_WEBHOOK_SECRET=whsec_tua_chiave_webhook

# Email SMTP
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=tua_email@gmail.com
MAIL_PASSWORD=tua_app_password
```

### 5. Configura il web server

#### Apache

Assicurati che il modulo rewrite sia abilitato:

```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

Configura il virtual host:

```apache
<VirtualHost *:80>
    ServerName qr-finder.com
    DocumentRoot /var/www/html/qr-finder/public
    
    <Directory /var/www/html/qr-finder/public>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/qr-finder-error.log
    CustomLog ${APACHE_LOG_DIR}/qr-finder-access.log combined
</VirtualHost>
```

#### Nginx

```nginx
server {
    listen 80;
    server_name qr-finder.com;
    root /var/www/html/qr-finder/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### 6. Configura Stripe Webhook

Nel dashboard di Stripe, configura un webhook che punti a:

```
https://qr-finder.com/api/webhooks/stripe
```

Seleziona gli eventi:
- `payment_intent.succeeded`
- `payment_intent.payment_failed`

### 7. Permessi

Assicurati che le directory di storage siano scrivibili:

```bash
chmod -R 755 storage/
chown -R www-data:www-data storage/
```

### 8. Cron Job (opzionale)

Per pulire le sessioni scadute, aggiungi un cron job:

```bash
# Pulisci sessioni ogni giorno alle 3 AM
0 3 * * * cd /var/www/html/qr-finder && php -r "require 'vendor/autoload.php'; \$config = require 'config/config.php'; \$db = new QrFinder\Utils\Database(\$config['database']); \$db->execute('DELETE FROM user_sessions WHERE expires_at < NOW()');"
```

## Struttura del Progetto

```
qr-finder/
├── config/
│   └── config.php          # Configurazione principale
├── database/
│   └── schema.sql          # Schema del database
├── public/
│   ├── index.php           # Entry point
│   └── .htaccess           # Configurazione Apache
├── src/
│   ├── Controllers/        # Controller MVC
│   ├── Models/             # Modello dati
│   ├── Services/           # Servizi (Email, Stripe, QR)
│   ├── Utils/              # Utility (Database)
│   └── Router.php          # Router
├── templates/              # Template HTML
├── storage/
│   └── qr_codes/           # QR code generati
├── assets/
│   ├── css/                # Fogli di stile
│   └── js/                 # JavaScript
├── composer.json           # Dipendenze PHP
└── .env                    # Variabili d'ambiente
```

## API Endpoints

### Autenticazione
- `POST /api/auth/register` - Registrazione
- `POST /api/auth/login` - Login
- `POST /api/auth/logout` - Logout
- `GET /api/auth/me` - Utente corrente
- `GET /api/auth/verify-email` - Verifica email
- `POST /api/auth/forgot-password` - Richiesta reset password
- `POST /api/auth/reset-password` - Reset password

### Oggetti
- `GET /api/objects` - Lista oggetti
- `POST /api/objects` - Crea oggetto
- `GET /api/objects/{id}` - Dettaglio oggetto
- `PUT /api/objects/{id}` - Aggiorna oggetto
- `DELETE /api/objects/{id}` - Elimina oggetto
- `POST /api/objects/{id}/payment-intent` - Crea pagamento Stripe
- `GET /api/objects/{id}/download` - Scarica QR code
- `GET /api/objects/{id}/map` - Dati mappa oggetto

### Notifiche
- `GET /api/notifications` - Notifiche non lette
- `GET /api/notifications/all` - Tutte le notifiche
- `GET /api/notifications/unread-count` - Conteggio notifiche
- `POST /api/notifications/{id}/read` - Segna come letta
- `POST /api/notifications/read-all` - Segna tutte come lette

### Pagamenti
- `POST /api/webhooks/stripe` - Webhook Stripe
- `GET /api/payments/history` - Storico pagamenti
- `GET /api/payments/{id}/status` - Stato pagamento

### Tracciamento (Pubblico)
- `GET /r/{shortCode}` - Pagina tracciamento
- `POST /r/{shortCode}` - Registra scansione
- `POST /r/{shortCode}/contact` - Contatta proprietario

## Sicurezza

- Password hashate con bcrypt
- JWT per l'autenticazione
- CSRF protection
- SQL injection prevention con prepared statements
- XSS protection con output escaping
- Rate limiting raccomandato (da implementare con fail2ban o simile)

## Licenza

MIT License

## Supporto

Per supporto o domande, contatta support@qr-finder.com
