
# **TP : Introduction à MVC & API REST avec Slim + Medoo (PHP)**

### *Gestion d’un mini coffre-fort numérique (upload, listing, suppression de fichiers)*

---

# 🎯 **Objectifs du TP**

À la fin de cette séance, vous serez capables de :

1. **Expliquer les principes d’une API REST**
   (ressource, routes, JSON, verbes HTTP, statuts…).

2. **Installer et utiliser un framework PHP minimaliste : Slim**
   (structure MVC simplifiée, routes, contrôleurs).

3. **Utiliser Medoo**, un micro-ORM simple pour interagir avec une base MySQL.

4. **Créer une API REST fonctionnelle** capable de :

   * Envoyer un fichier (upload)
   * Lister tous les fichiers
   * Afficher les informations d’un fichier
   * Télécharger un fichier
   * Supprimer un fichier
   * Consulter des statistiques (taille totale / quota)

5. **Tester correctement une API avec Postman**.

6. Comprendre comment cette API pourra être utilisée plus tard
   → par un client JavaFX ou une interface web.

---

# 🧠 **PARTIE 1 — Comprendre REST (cours)**

## 1.1 Qu’est-ce qu’une API ?

Une API est une interface permettant à deux programmes de communiquer entre eux.
Ici : un serveur **PHP** communiquera avec un client **JavaFX** ou une appli web.

## 1.2 Qu’est-ce qu’une API REST ?

REST repose sur 4 idées principales :

### 🔹 1. **Une ressource**

C’est un type d’objet que l’on manipule.

Dans ce TP, notre ressource s’appelle **File** (un fichier).

### 🔹 2. **Une URI** (l’adresse de la ressource)

Exemples :

| Action              | Méthode | URI                    |
| ------------------- | ------- | ---------------------- |
| Lister les fichiers | GET     | `/files`               |
| Voir un fichier     | GET     | `/files/{id}`          |
| Uploader un fichier | POST    | `/files`               |
| Télécharger         | GET     | `/files/{id}/download` |
| Supprimer           | DELETE  | `/files/{id}`          |

### 🔹 3. **Des verbes HTTP**

* `GET` → lire
* `POST` → créer
* `PUT/PATCH` → modifier
* `DELETE` → supprimer

### 🔹 4. **Une réponse JSON**

Exemple :

```json
{
  "id": 1,
  "filename": "test.pdf",
  "size": 23456
}
```

### 🔹 5. **Des statuts HTTP**

* `200 OK` → tout va bien
* `201 Created` → ressource créée
* `400 Bad Request` → mauvaise requête
* `404 Not Found` → ressource inexistante
* `500 Internal Server Error` → erreur serveur

---

# 💻 **PARTIE 2 — Mise en place du projet Slim**

## 2.1 Installation du projet

### 👉 Étape 1 : créer un dossier de projet

Appelez-le :

```
file-vault-api
```

### 👉 Étape 2 : initialiser Composer

Dans le dossier :

```
composer init
```

(Pressez ENTER pour toutes les questions.)

### 👉 Étape 3 : installer Slim + Medoo

```
composer require slim/slim:"^4.0" slim/psr7 catfan/medoo
```

---

## 2.2 Structure du projet

Créez les dossiers suivants :

```
file-vault-api/
│── public/
│    └── index.php
│── src/
│    ├── Controller/
│    │      └── FileController.php
│    └── Model/
│           └── FileRepository.php
└── storage/
        └── uploads/
```

Le dossier **uploads** servira à stocker les fichiers envoyés.

---

## 2.3 Création de la base de données

Dans phpMyAdmin / MySQL :

```sql
CREATE DATABASE file_vault;
USE file_vault;

CREATE TABLE files (
    id INT AUTO_INCREMENT PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    stored_name VARCHAR(255) NOT NULL,
    size BIGINT NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    uploaded_at DATETIME NOT NULL
);

CREATE TABLE settings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    value VARCHAR(255) NOT NULL
);

INSERT INTO settings (name, value) VALUES ('quota_bytes', '52428800'); -- 50 Mo
```

---

# 🚀 **PARTIE 3 — Démarrage du framework Slim**

Créez le fichier :

## `public/index.php`

Collez :

```php
<?php
use Slim\Factory\AppFactory;
use Medoo\Medoo;
use App\Controller\FileController;

require __DIR__ . '/../vendor/autoload.php';

$database = new Medoo([
    'type' => 'mysql',
    'host' => '127.0.0.1',
    'database' => 'file_vault',
    'username' => 'root',
    'password' => '',
]);

$app = AppFactory::create();

$app->addBodyParsingMiddleware();

$fileController = new FileController($database);

$app->get('/files', [$fileController, 'list']);
$app->get('/files/{id}', [$fileController, 'show']);
$app->get('/files/{id}/download', [$fileController, 'download']);
$app->post('/files', [$fileController, 'upload']);
$app->delete('/files/{id}', [$fileController, 'delete']);
$app->get('/stats', [$fileController, 'stats']);

$app->run();
```

---

# 🏛 **PARTIE 4 — Le “M” de MVC : le Model (Medoo)**

Créez le fichier :

## `src/Model/FileRepository.php`

```php
<?php
namespace App\Model;

use Medoo\Medoo;

class FileRepository
{
    private Medoo $db;

    public function __construct(Medoo $db)
    {
        $this->db = $db;
    }

    public function listFiles(): array
    {
        return $this->db->select('files', '*');
    }

    public function find(int $id): ?array
    {
        return $this->db->get('files', '*', ['id' => $id]) ?: null;
    }

    public function create(array $data): int
    {
        $this->db->insert('files', $data);
        return (int)$this->db->id();
    }

    public function delete(int $id): void
    {
        $this->db->delete('files', ['id' => $id]);
    }

    public function totalSize(): int
    {
        return (int)$this->db->sum('files', 'size') ?: 0;
    }

    public function quotaBytes(): int
    {
        return (int)$this->db->get('settings', 'value', ['name' => 'quota_bytes']);
    }
}
```

### ✔️ **Exercice Étudiant A :**

Ajouter une méthode :

```php
public function countFiles(): int
```

---

# 🎮 **PARTIE 5 — Le “C” de MVC : le Controller (Slim)**

Créez :

## `src/Controller/FileController.php`
```php
<?php
// src/Controller/FileController.php

namespace App\Controller;

use App\Model\FileRepository;
use Medoo\Medoo;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

class FileController
{
    private FileRepository $files;
    private string $uploadDir;

    public function __construct(Medoo $db)
    {
        $this->files = new FileRepository($db);
        $this->uploadDir = __DIR__ . '/../../storage/uploads';
    }

    // GET /files
    public function list(Request $request, Response $response): Response
    {
        $data = $this->files->listFiles();

        $payload = json_encode($data, JSON_PRETTY_PRINT);
        $response->getBody()->write($payload);
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus(200);
    }

    // GET /files/{id}
    public function show(Request $request, Response $response, array $args): Response
    {
        $id = (int)$args['id'];
        $file = $this->files->find($id);

        if (!$file) {
            $response->getBody()->write(json_encode(['error' => 'File not found']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(404);
        }

        $response->getBody()->write(json_encode($file, JSON_PRETTY_PRINT));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(200);
    }

    // POST /files  (upload via form-data)
    public function upload(Request $request, Response $response): Response
    {
        $uploadedFiles = $request->getUploadedFiles();

        if (!isset($uploadedFiles['file'])) {
            $response->getBody()->write(json_encode(['error' => 'No file uploaded']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(400);
        }

        $file = $uploadedFiles['file'];

        if ($file->getError() !== UPLOAD_ERR_OK) {
            $response->getBody()->write(json_encode(['error' => 'Upload error']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(400);
        }

        $size = $file->getSize();
        $totalSize = $this->files->totalSize();
        $quota = $this->files->quotaBytes();

        if ($quota > 0 && ($totalSize + $size) > $quota) {
            $response->getBody()->write(json_encode(['error' => 'Quota exceeded']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(413);
        }

        $originalName = $file->getClientFilename();
        $mimeType = $file->getClientMediaType();
        $storedName = uniqid('f_', true) . '_' . $originalName;

        $file->moveTo($this->uploadDir . DIRECTORY_SEPARATOR . $storedName);

        $id = $this->files->create([
            'filename'    => $originalName,
            'stored_name' => $storedName,
            'size'        => $size,
            'mime_type'   => $mimeType,
            'uploaded_at' => date('Y-m-d H:i:s')
        ]);

        $response->getBody()->write(json_encode([
            'message' => 'File uploaded',
            'id'      => $id
        ], JSON_PRETTY_PRINT));

        return $response->withHeader('Content-Type', 'application/json')->withStatus(201);
    }

    // GET /files/{id}/download
    public function download(Request $request, Response $response, array $args): Response
    {
        $id = (int)$args['id'];
        $file = $this->files->find($id);

        if (!$file) {
            $response->getBody()->write('File not found');
            return $response->withStatus(404);
        }

        $path = $this->uploadDir . DIRECTORY_SEPARATOR . $file['stored_name'];

        if (!file_exists($path)) {
            $response->getBody()->write('File missing on disk');
            return $response->withStatus(500);
        }

        $stream = fopen($path, 'rb');
        $response->getBody()->write(stream_get_contents($stream));
        fclose($stream);

        return $response
            ->withHeader('Content-Type', $file['mime_type'])
            ->withHeader('Content-Disposition', 'attachment; filename="' . $file['filename'] . '"')
            ->withStatus(200);
    }

    // DELETE /files/{id}
    public function delete(Request $request, Response $response, array $args): Response
    {
        $id = (int)$args['id'];
        $file = $this->files->find($id);

        if (!$file) {
            $response->getBody()->write(json_encode(['error' => 'File not found']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(404);
        }

        // Supprimer le fichier sur le disque
        $path = $this->uploadDir . DIRECTORY_SEPARATOR . $file['stored_name'];
        if (file_exists($path)) {
            unlink($path);
        }

        // Supprimer en base
        $this->files->delete($id);

        $response->getBody()->write(json_encode(['message' => 'File deleted']));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(200);
    }

    // GET /stats
    public function stats(Request $request, Response $response): Response
    {
        $totalSize = $this->files->totalSize();
        $quota = $this->files->quotaBytes();

        // Exercice 1: utiliser countFiles() ici si l'étudiant l’a codée
        // $count = $this->files->countFiles();

        $data = [
            'total_size_bytes' => $totalSize,
            'quota_bytes'      => $quota,
            // 'file_count'        => $count,
        ];

        $response->getBody()->write(json_encode($data, JSON_PRETTY_PRINT));
        return $response->withHeader('Content-Type', 'application/json')->withStatus(200);
    }
}
```                                      

➡️ Ce fichier contient toutes les routes REST :

* Upload
* Listing
* Affichage
* Suppression
* Statistiques
* Téléchargement

---

# 🧪 **PARTIE 6 — Tester votre API avec Postman**

## 💡 Votre serveur doit être accessible via :

```
http://localhost/file-vault-api/public
```

---

## 6.1 Tester : GET /files

* Méthode : **GET**
* URL : `/files`
* Résultat attendu : un tableau JSON.

---

## 6.2 Tester : POST /files (upload)

* Méthode : **POST**
* URL : `/files`
* Onglet **Body** :

  * Type : **form-data**
  * Clé : `file` (type `File`)
  * Envoyez un PDF / PNG / TXT

Résultat attendu :

```json
{
  "message": "File uploaded",
  "id": 1
}
```

---

## 6.3 Tester : GET /files/{id}

* Méthode : **GET**
* Exemple : `/files/1`

---

## 6.4 Tester : GET /files/{id}/download`

Télécharge le fichier.

---

## 6.5 Tester : DELETE /files/{id}

Supprime définitivement :

```json
{ "message": "File deleted" }
```

---

## 6.6 Tester : GET /stats

Affiche la taille totale + quota :

```json
{
  "total_size_bytes": 125000,
  "quota_bytes": 52428800
}
```

### ✔️ **Exercice Étudiant B :**

Ajouter dans `/stats` :

* le nombre total de fichiers → `countFiles()`
* le pourcentage d’utilisation du quota

---

# 🧩 **PARTIE 7 — Travail final demandé**

Vous devez avoir :

### ✅ Une API Slim fonctionnelle

avec les routes :

| Route                    | Action         |
| ------------------------ | -------------- |
| GET /files               | liste          |
| GET /files/{id}          | détail         |
| POST /files              | upload         |
| GET /files/{id}/download | téléchargement |
| DELETE /files/{id}       | suppression    |
| GET /stats               | statistiques   |

### ✅ Une base de données fonctionnelle

### ✅ Un test complet sur Postman

→ capture d’écran attendue pour chaque opération.

### 📌 Bonus (facultatif mais recommandé)

* Ajouter une route PUT `/quota` pour modifier le quota global.
* Empêcher l’upload de fichiers dont certaines extensions ne sont pas autorisées.
* Ajouter une pagination dans `GET /files`.

---

# 🎓 **Fin du TP — Ce que vous devez retenir**

* **Slim** = mini framework très simple → parfait pour comprendre MVC.
* **Medoo** = micro-ORM → syntaxe ultra simple, idéal pour débuter.
* **REST** = approche moderne pour exposer des services.
* Une API REST peut être consommée par :

  * une appli JavaFX (client lourd),
  * une interface web (client léger),
  * Postman,
  * d’autres services.
