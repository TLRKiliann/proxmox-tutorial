# Projects PostgreSQL with another CTX with Node.js

## PostgreSQL installation

Use template: `tunkey-postgresql`

`pct enter <CT-ID>`

`hostname -I`

port: 5432

`telnet 192.168.18.<CT-ID> 5432`

```
nano /etc/postgresql/*/main/postgresql.conf

listen_addresses = '*'
```

```
nano /etc/postgresql/*/main/pg_hba.conf

# Autoriser votre conteneur Node.js à se connecter
host    all             all             192.168.xx.<CT-ID>/24            md5
```

`systemctl restart postgresql`


## Depuis le conteneur PostgreSQL

`su - postgres`

`psql`

```
-- Créer la table utilisateurs
CREATE TABLE IF NOT EXISTS utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Vérifier que la table est créée
\dt

-- (Optionnel) Ajouter un utilisateur de test
INSERT INTO utilisateurs (nom, email) VALUES ('Test User', 'test@example.com');

-- Quitter psql
\q
```

`systemctl status postgresql`

`systemctl restart postgresql`

```
# Éditer postgresql.conf
nano /etc/postgresql/*/main/postgresql.conf

# Éditer pg_hba.conf
nano /etc/postgresql/*/main/pg_hba.conf
```

- Connectez-vous sans mot de passe dans le conteneur avec 

`su - postgres -c psql`

- Définissez un nouveau mot de passe avec la commande `\password` dans psql :


`ALTER USER postgres WITH PASSWORD 'VotreNouveauMotDePasse';`


Utilisez ce nouveau mot de passe dans votre app.js

```
App.js (avec module pg)

const { Client } = require('pg');

// Remplacez par les bonnes valeurs
const client = new Client({
  user: 'votre_utilisateur',
  password: 'VotreNouveauMotDePasse',
  host: 'IP_DU_CONTENEUR_POSTGRESQL', // L'adresse notée à l'Étape 1
  port: 5432,
  database: 'votre_base_de_donnees'
});

client.connect();

```

et lancer: `node app.js`