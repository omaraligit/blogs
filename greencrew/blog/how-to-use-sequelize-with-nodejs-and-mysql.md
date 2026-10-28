# How to Use Sequelize with Node.js and MySQL (Models, Migrations, Associations, and Queries)

Author: [greencrew](https://greencrew.space/blog)

Tags: Node.js, Sequelize, MySQL, ORM, JavaScript, DevOps

Description: To use Sequelize with Node.js and MySQL, run npm install sequelize mysql2, create a connection with new Sequelize(database, user, password, { host: 'localhost', dialect: 'mysql' }), check it with sequelize.authenticate(), and define models with sequelize.define(). This guide covers creating the MySQL user, sequelize-cli migrations and seeders, CRUD queries, operators, associations, transactions, and common connection errors.

Date: 2026-10-28

**Short answer:** Install the ORM and the MySQL driver with `npm install sequelize mysql2`. Connect with `new Sequelize('appdb', 'appuser', 'password', { host: 'localhost', dialect: 'mysql' })` and verify the connection with `await sequelize.authenticate()`. Define tables as models (`sequelize.define('Book', { title: DataTypes.STRING })`), and manage schema changes with `sequelize-cli` migrations (`npx sequelize-cli db:migrate`) instead of `sync()` in production. Query with `Book.findAll({ where: { ... } })`, link tables with `hasMany`/`belongsTo`, and wrap multi-step writes in `sequelize.transaction()`.

## What Is Sequelize?

[Sequelize](https://sequelize.org/) is a promise-based ORM (Object-Relational Mapper) for Node.js. It maps database tables to JavaScript classes, so you write `Book.findByPk(1)` instead of raw SQL. It supports MySQL, MariaDB, PostgreSQL, SQLite, and Microsoft SQL Server. This guide uses Sequelize v6, the stable release line documented at [sequelize.org/docs/v6](https://sequelize.org/docs/v6/). The next major is published separately as `@sequelize/core`.

| Concept | What it is | Example |
| --- | --- | --- |
| Instance | The connection and its pool | `new Sequelize(...)` |
| Model | A table | `Book` → `Books` table |
| Attribute | A column | `title: DataTypes.STRING` |
| Migration | A versioned schema change | `20261028-create-book.js` |
| Seeder | Sample or reference data | `20261028-demo-books.js` |
| Association | A relationship between tables | `Author.hasMany(Book)` |

## Prerequisites

- Node.js LTS installed ([how to install Node.js on Ubuntu](/blog/how-to-install-nodejs-on-ubuntu))
- MySQL or MariaDB running locally or on a server. Installation and `mysql_secure_installation` are covered in [how to install a LEMP stack on Ubuntu](/blog/how-to-install-lemp-stack-on-ubuntu).

## Step 1: Create the MySQL Database and User

Never connect your app as MySQL `root`. Create a dedicated user:

```bash
sudo mysql <<'SQL'
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'change-this-strong-password';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
SQL
```

## Step 2: Install Sequelize and Connect to MySQL

```bash
mkdir books-api && cd books-api
npm init -y
npm install sequelize mysql2 dotenv
```

`mysql2` is the driver Sequelize uses for MySQL. Use `mariadb` with `dialect: 'mariadb'` if you're on MariaDB and prefer its native driver.

Create `.env` (and add it to `.gitignore`):

```bash
DB_HOST=127.0.0.1
DB_NAME=appdb
DB_USER=appuser
DB_PASS=change-this-strong-password
```

Create `db.js`:

```javascript
require('dotenv').config();
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(process.env.DB_NAME, process.env.DB_USER, process.env.DB_PASS, {
  host: process.env.DB_HOST,
  dialect: 'mysql',
  logging: process.env.NODE_ENV === 'production' ? false : console.log,
  pool: { max: 10, min: 0, acquire: 30000, idle: 10000 },
});

module.exports = sequelize;
```

Test it:

```javascript
// check-db.js
const sequelize = require('./db');

(async () => {
  try {
    await sequelize.authenticate();
    console.log('Connected to MySQL');
  } catch (err) {
    console.error('Unable to connect:', err.message);
  } finally {
    await sequelize.close();
  }
})();
```

```bash
node check-db.js
```

| Pool option | Meaning |
| --- | --- |
| `max` | Maximum open connections per Node process. Multiply by the number of processes when you [cluster](/blog/how-to-scale-nodejs-applications-with-clustering), and stay below MySQL's `max_connections`. |
| `min` | Connections kept open when idle |
| `acquire` | Milliseconds to wait for a free connection before throwing |
| `idle` | Milliseconds before an idle connection is released |

## Step 3: Define a Model

```javascript
// models/book.js
const { DataTypes } = require('sequelize');
const sequelize = require('../db');

const Book = sequelize.define('Book', {
  title: { type: DataTypes.STRING, allowNull: false },
  author: { type: DataTypes.STRING, allowNull: false },
  releaseDate: { type: DataTypes.DATEONLY },
  subject: { type: DataTypes.INTEGER },
}, {
  tableName: 'books',     // otherwise Sequelize pluralizes to "Books"
  underscored: true,      // created_at / release_date column names
});

module.exports = Book;
```

| Option | Default | Effect |
| --- | --- | --- |
| `timestamps` | `true` | Adds and manages `createdAt` / `updatedAt` |
| `tableName` | Pluralized model name | Exact table name |
| `freezeTableName` | `false` | Use the model name as-is |
| `underscored` | `false` | `snake_case` column names |
| `paranoid` | `false` | Soft deletes with `deletedAt` (needs timestamps) |

Common data types: `STRING` (VARCHAR 255), `TEXT`, `INTEGER`, `BIGINT`, `DECIMAL(10, 2)`, `BOOLEAN`, `DATE` (DATETIME), `DATEONLY`, `JSON`, and `ENUM('draft', 'published')`.

### sync() vs Migrations

`await sequelize.sync()` creates missing tables from your models, which is handy for a quick prototype. `sync({ alter: true })` and `sync({ force: true })` change or drop tables to match the models. Never run those against production data. For anything real, use migrations.

## Step 4: Manage the Schema with sequelize-cli Migrations

```bash
npm install --save-dev sequelize-cli
npx sequelize-cli init
```

This creates `config/config.json`, `models/`, `migrations/`, and `seeders/`. Swap the JSON config for a JS file so it reads environment variables. Create `.sequelizerc`:

```javascript
const path = require('node:path');
module.exports = { config: path.resolve('config', 'config.js') };
```

```javascript
// config/config.js
require('dotenv').config();
const base = {
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  host: process.env.DB_HOST,
  dialect: 'mysql',
};
module.exports = { development: base, test: base, production: { ...base, logging: false } };
```

Generate a model and its migration:

```bash
npx sequelize-cli model:generate --name Author --attributes name:string,country:string
npx sequelize-cli model:generate --name Book --attributes title:string,releaseDate:dateonly,authorId:integer
```

Add a foreign key to the generated Book migration's `authorId` column:

```javascript
authorId: {
  type: Sequelize.INTEGER,
  allowNull: false,
  references: { model: 'Authors', key: 'id' },
  onDelete: 'CASCADE',
},
```

Run and inspect migrations:

| Command | What it does |
| --- | --- |
| `npx sequelize-cli db:migrate` | Apply pending migrations |
| `npx sequelize-cli db:migrate:status` | Show applied and pending migrations |
| `npx sequelize-cli db:migrate:undo` | Roll back the last migration |
| `npx sequelize-cli migration:generate --name add-isbn-to-books` | Empty migration for a schema change |
| `npx sequelize-cli seed:generate --name demo-books` | New seeder file |
| `npx sequelize-cli db:seed:all` | Run all seeders |
| `NODE_ENV=production npx sequelize-cli db:migrate` | Use the production config block |

Sequelize records applied migrations in a `SequelizeMeta` table, so each one runs exactly once per database. Run `db:migrate` as a deploy step before restarting the app.

## Step 5: Query Data (CRUD)

The CLI-generated `models/index.js` loads every model and runs its `associate` function. With the models from Step 4:

```javascript
const { Author, Book } = require('./models');
const { Op } = require('sequelize');

// Create
const author = await Author.create({ name: 'Ursula K. Le Guin', country: 'US' });
await Book.create({ title: 'The Dispossessed', releaseDate: '1974-05-01', authorId: author.id });

// Read
const all = await Book.findAll();
const one = await Book.findByPk(1);
const recent = await Book.findAll({
  where: { releaseDate: { [Op.gte]: '1970-01-01' } },
  order: [['releaseDate', 'DESC']],
  limit: 10,
  attributes: ['id', 'title', 'releaseDate'],
});

// Update
await Book.update({ title: 'The Dispossessed: An Ambiguous Utopia' }, { where: { id: 1 } });

// Delete
await Book.destroy({ where: { id: 1 } });
```

| Operator | SQL | Example |
| --- | --- | --- |
| `Op.eq` / `Op.ne` | `=` / `!=` | `{ status: { [Op.ne]: 'draft' } }` |
| `Op.gt`, `Op.gte`, `Op.lt`, `Op.lte` | `>`, `>=`, `<`, `<=` | `{ price: { [Op.lt]: 20 } }` |
| `Op.in` | `IN (...)` | `{ id: { [Op.in]: [1, 2, 3] } }` |
| `Op.like` | `LIKE` | `{ title: { [Op.like]: '%dragon%' } }` |
| `Op.between` | `BETWEEN` | `{ year: { [Op.between]: [1960, 1980] } }` |
| `Op.or` | `OR` | `{ [Op.or]: [{ a: 1 }, { b: 2 }] }` |

Paginate with `findAndCountAll({ limit, offset })`. When you need raw SQL, use bound parameters, never string concatenation: `sequelize.query('SELECT * FROM Books WHERE id = ?', { replacements: [id], type: QueryTypes.SELECT })`.

## Step 6: Add Associations

In `models/author.js` and `models/book.js`, fill in the generated `associate` methods:

```javascript
// models/author.js
static associate(models) {
  Author.hasMany(models.Book, { foreignKey: 'authorId', as: 'books' });
}

// models/book.js
static associate(models) {
  Book.belongsTo(models.Author, { foreignKey: 'authorId', as: 'author' });
}
```

Load related rows with `include` (an SQL JOIN):

```javascript
const authors = await Author.findAll({
  include: [{ model: Book, as: 'books', attributes: ['title'] }],
});
```

| Relationship | Methods | Foreign key lives on |
| --- | --- | --- |
| One-to-one | `hasOne` + `belongsTo` | The `belongsTo` side |
| One-to-many | `hasMany` + `belongsTo` | The "many" side |
| Many-to-many | `belongsToMany` on both, `{ through: 'BookGenres' }` | A junction table |

Loading associations inside a loop (`for (const a of authors) await a.getBooks()`) runs one query per row. Use `include` to load everything in one query.

## Step 7: Use Transactions for Multi-Step Writes

```javascript
const { sequelize } = require('./models');

await sequelize.transaction(async (t) => {
  const author = await Author.create({ name: 'N. K. Jemisin' }, { transaction: t });
  await Book.create({ title: 'The Fifth Season', authorId: author.id }, { transaction: t });
}); // commits if the callback resolves, rolls back if it throws
```

Pass `{ transaction: t }` to every query inside the callback. A query without it runs outside the transaction.

## Troubleshooting Sequelize and MySQL

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Please install mysql2 package manually` | Driver not installed | `npm install mysql2` |
| `SequelizeConnectionRefusedError: connect ECONNREFUSED 127.0.0.1:3306` | MySQL not running or listening elsewhere | `sudo systemctl status mysql`, check host and port |
| `SequelizeAccessDeniedError: Access denied for user` | Wrong password, or user created for another host | Recreate `'appuser'@'localhost'`. Match `DB_HOST`. |
| `Unknown column 'createdAt' in 'field list'` | Table created without timestamps | Add the columns in a migration, or `timestamps: false` |
| `Table 'appdb.Books' doesn't exist` | Pluralized table name vs actual name | Set `tableName`, or run your migrations |
| `SequelizeConnectionAcquireTimeoutError` | Pool exhausted (slow queries or leaked transactions) | Raise `pool.max` carefully, fix slow queries, always finish transactions |
| `ER_CON_COUNT_ERROR: Too many connections` | `pool.max` × processes > `max_connections` | Lower `pool.max` or raise MySQL's limit |
| Migration ran in the wrong database | `NODE_ENV` not set | `NODE_ENV=production npx sequelize-cli db:migrate` |
| Emoji saved as `????` | Not using `utf8mb4` | Create the database and tables with `utf8mb4` |

## Monitor the API and Its Database Connection

Most Sequelize outages show up as a hung pool or refused connections, and the Node process stays up while every request fails. Add a health route that actually touches the database:

```javascript
app.get('/health', async (req, res) => {
  try {
    await sequelize.query('SELECT 1');
    res.json({ status: 'ok' });
  } catch {
    res.status(503).json({ status: 'db-down' });
  }
});
```

Then point [GreenCrew](https://greencrew.space) at it. GreenCrew runs HTTP uptime checks and alerts you on Discord, Slack, or Telegram when the endpoint stops responding correctly, with a public status page for your users. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I use sequelize.sync() in production?

No. `sync()` can't express renames or data changes, and `alter`/`force` can drop data. Use `sequelize-cli` migrations so every schema change is versioned, reviewed, and reversible.

### Does Sequelize work with MariaDB?

Yes. Use `dialect: 'mariadb'` with the `mariadb` driver package, or keep `dialect: 'mysql'` with `mysql2`, which also works against MariaDB for most apps.

### How do I prevent SQL injection with Sequelize?

Model methods (`findAll`, `create`, `update` with `where`) escape values for you. With `sequelize.query`, always use `replacements` or `bind` parameters, and never interpolate user input into the SQL string.

### Sequelize or Prisma or TypeORM?

All three work with MySQL. Sequelize is mature, JavaScript-first, and has a large ecosystem. Prisma uses a schema file and a generated, strongly typed client. TypeORM fits decorator-based TypeScript projects such as NestJS. Pick the one that matches your codebase and team.

## Conclusion

Using Sequelize with Node.js and MySQL comes down to a few steps. Install `sequelize` and `mysql2`, connect with a dedicated MySQL user loaded from environment variables, define models, and evolve the schema with `sequelize-cli` migrations instead of `sync()`. Use operators and `include` for queries, transactions for multi-step writes, and size your pool for the number of Node processes. Then deploy the API behind [PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) or [Docker](/blog/how-to-build-a-nodejs-application-with-docker), and let [GreenCrew](https://greencrew.space) watch a database-backed `/health` route.
