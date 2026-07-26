# Database — VeloraSec

> Skema dan migrasi database MySQL untuk platform VeloraSec.

---

## Status

🔴 **Belum Dikembangkan** — Ini adalah placeholder untuk **Phase 8 (Database MySQL)**.

---

## Rencana Teknologi

| Komponen | Teknologi |
|---|---|
| Database Engine | MySQL 8.0+ |
| ORM | SQLAlchemy (via Flask-SQLAlchemy) |
| Migration | Flask-Migrate (Alembic) |
| Charset | utf8mb4 (full Unicode + emoji) |
| Collation | utf8mb4_unicode_ci |

---

## Rencana Skema Database

### Tabel: `users`
```sql
CREATE TABLE users (
    id            INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username      VARCHAR(50)  NOT NULL UNIQUE,
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    is_active     BOOLEAN      DEFAULT TRUE,
    is_admin      BOOLEAN      DEFAULT FALSE,
    created_at    DATETIME     DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Tabel: `user_progress`
```sql
CREATE TABLE user_progress (
    id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id      INT UNSIGNED NOT NULL,
    module_id    VARCHAR(50)  NOT NULL,
    is_completed BOOLEAN      DEFAULT FALSE,
    completed_at DATETIME     NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uq_user_module (user_id, module_id)
);
```

### Tabel: `quiz_results`
```sql
CREATE TABLE quiz_results (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id    INT UNSIGNED NOT NULL,
    category   VARCHAR(100) NOT NULL,
    score      TINYINT      NOT NULL,
    total      TINYINT      NOT NULL,
    taken_at   DATETIME     DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### Tabel: `planner_tasks`
```sql
CREATE TABLE planner_tasks (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id    INT UNSIGNED NOT NULL,
    task_key   VARCHAR(20)  NOT NULL,   -- format: w0d0, w1d3, dst.
    is_done    BOOLEAN      DEFAULT FALSE,
    updated_at DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uq_user_task (user_id, task_key)
);
```

---

## Rencana Struktur Folder

```
database/
├── README.md              ← file ini
├── schema.sql             → DDL lengkap semua tabel
├── seed_data.sql          → data awal (opsional)
└── migrations/            → file migrasi Alembic (auto-generated)
```

---

## Setup Database (Setelah Phase 8 Dimulai)

```sql
-- 1. Buat database
CREATE DATABASE velorasec
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- 2. Buat user
CREATE USER 'velorasec_user'@'localhost' IDENTIFIED BY 'strong_password_here';
GRANT ALL PRIVILEGES ON velorasec.* TO 'velorasec_user'@'localhost';
FLUSH PRIVILEGES;
```

```bash
# 3. Jalankan migrasi via Flask-Migrate
cd backend
flask db upgrade
```

---

*Lihat [Backend README](../backend/README.md) untuk detail ORM dan endpoint API.*
