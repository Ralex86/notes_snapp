# Journal de bord

On doit réécrire une api `node.js` en ruby on rails. Celle ci gère les tickets de caisse des clients et sinsère dans un projet plus large de reconnaissance dimage ([OCR](https://fr.wikipedia.org/wiki/Reconnaissance_optique_de_caractères) = reconnaissance optique de caracteres).

## Sommaire

1. [Objectifs](#objectifs)

2. [Model](#model)

   [Ticket/command](#ticket/command)

   [Ticket/query](#ticket/query)

3. [Controller](#controller)

   [lib/web](#ticket/web)

4. [RubyOnRails](#rubyOnRails)

## Objectifs

Le pattern utilise dans le projet node est de type DDD [ Domain Driven Design ](https://en.wikipedia.org/wiki/Domain-driven**design). Il faudra changer ce pattern en MC (model controler) sans le V car il ny a pas de vues a gerer. La structure du projet suit aussi une architecture de type [CQRS](https://docs.microsoft.com/fr-fr/azure/architecture/guide/architecture-styles/cqrs) (command query responsability segregation). Les `command` et les `query` de laggregat **Ticket** classe de type **CRUD** creent des **Event** stackés dans un **Bus**. Ces **Event** sont traités par `web` qui manipule le SGBD.

> On retrouve ici un pattern de type event/payload/applyEvent/storeSubscribe qui me fait fortement penser a Redux...

1. Pour la comprehension du projet on se concentrera dabord sur la liste des use cases quon retrouvera dans les repertoires `lib/ticket/command/handler.js` et `lib/ticjet/query/handler`
2. Ensuite on verra comment acceder a lapi depuis lexterieur dans le dossier `/lib/web`
3. Enfin dans un dernier temps on commencera un nouveau projet ruby pour re implementer lapi

> On veillera a se concentrer sur la spec et sur la definition des fonctions métiers.

## Model

### ticket/command

Pour rappel `ticket.js` est un **agregated root** dans le language DDD. Cest un objet de type **CRUD** (create, read, update, delete). Cette entite na pas dautre metier que de persister. Les handlers sont limplementation concrete du CRUD. `ticket.js` ne fait que suivre le pattern du DDD avec le schéma **payload** et souscription a une liste d **events**

### Analyse

On charge les dependances du pattern _DDD_.

```
const {AggregateRoot, jsonEqual, Event} = require('ddd');
```

On type lobjet `Creation` avec `tcomb`. Cest une sorte de libraire de typage comme `flow` pour sassurer que chaque objet a le bon type et sil est optionnel (cest une chaine de caracteres `t.String()` et ce parametre est optionnel `t.maybe()`). Sa structure suit la convention **payload/messageType**

```javascript
const Creation = t.interface(
  {
    id: t.String,
    editedAt: t.maybe(t.Date),
    comment: t.maybe(t.String),
    paths: t.maybe(t.list(t.String)),
    ossIds: t.maybe(t.list(t.String)),
    customerId: t.String,
    loyaltyCardId: t.String,
  },
  'TicketCreation',
);
```

Un autre exemple avec `Update`

```javascript
const Update = t.interface(
  {
    editedAt: t.maybe(t.Date),
    comment: t.maybe(t.String),
  },
  'TicketUpdate',
);
```

On crée la classe `Ticket` qui herite des proprietes de `AggregateRoot`. Cette classe a la structure typique dun _CRUD_

```javascript
class Ticket extends AggregateRoot{

update(update) {
...
}

delete() {
...
}

static create(creation) {
Creation(creation);
...
}

static deleteCustomer(deletion) {
CustomerDeletion(deletion);
...
}
```

Prenons lexemple de la fonction `static create(creation)`

```javascript
  static create(creation) {
    Creation(creation);
    let ticket = AggregateRoot.bootstrap({type: Ticket, id: creation.id});
    let payload = _.omit(Object.assign({}, creation, {isDeleted: false}), 'id');
    let ticketCreatedEvent = ticket.createEvent('TicketCreated', payload);
    ticket.applyEvent(ticketCreatedEvent);
    return {object: ticket, events: [ticketCreatedEvent]};
  }
```

- Tout dabord on verifie le parametre `creation` avec `tcomb`
- on cree un _aggregat_ avec labstraction `AggregateRoot`
- on cree un _payload_
- on cree un evenement `ticketCreatedEvent` avec son _payload_ et son _messageType_
- enfin on retourne lobjet ticket et sa liste devenements

```javascript
return {object: ticket, events: [ticketCreatedEvent]};
```

### ticket/query

Regardons la classe TicketCreated

```javascript
class TicketCreated {
  constructor(creation) {
    let {database} = Creation(creation);
    this._database = database;
  }

  handle(message) {
    let {
      date,
      payload,
      aggregateRoot: {id},
    } = Message(message);
    let insert = {
      id,
      uploaded_at: date,
      edited_at: payload.editedAt || date,
      comment: payload.comment,
      customer_id: payload.customerId,
      loyalty_card_id: payload.loyaltyCardId,
      paths: payload.paths,
      oss_ids: payload.ossIds,
    };
    this._database.insert('tickets', insert);
  }

  get messageType() {
    return 'TicketCreated';
  }
}
```

- le constructeur initialise lobjet membre `database` avec le wrapper `knexx` dans le package DDD
- on a une methode `handle(message)` pour traiter levenement en question ici: `'TicketCreated'`
- on a aussi une methode `messageType` pour savoir quel est le nom de levenement
- enfin on traite l'insertion dans la table `tickets` avec le payload destructuré de lobjet retourne par la fonction `Message`

> `lib/ticket` joue le role du **modele**

## Controller

### lib/web

Cette partie implemente la partie routage de lapi avec le framework Express.

- Le fichier `server.js` configure les middlewares, notament celui de lauthentication.
- Le fichier `route.js` gère les routes de lapi avec un router et instancie les differentes methodes (GET, POST, DELETE...)

> route joue le role du **controleur**

```javascript
function createRouter(creation) {
  let {
    infrastructure: {
      commandBus,
      queryBus,
      services: {fileServiceFactory},
    },
  } = creation;
  let router = new ExpressRouter();
  router.get('/admin/ping', (request, response) => response.send('OK'));
  router.post('/tickets', (...args) =>
    new TicketsPost({commandBus}).handle(...args),
  );
  router.get('/tickets', (...args) =>
    new TicketsGet({queryBus}).handle(...args),
  );
  router.get('/tickets/:id/read/:index', (...args) =>
    new TicketReadGet({queryBus, fileServiceFactory}).handle(...args),
  );
  router.put('/tickets/:id', (...args) =>
    new TicketPut({commandBus}).handle(...args),
  );
  router.delete('/tickets/:id', (...args) =>
    new TicketDelete({commandBus}).handle(...args),
  );
  router.get('/tickets/count', (...args) =>
    new TicketCountGet({queryBus}).handle(...args),
  );
  router.delete('/customers/:id', (...args) =>
    new CustomerDelete({commandBus}).handle(...args),
  );
  return router;
}
```

## RubyOnRails

Le but maintenant est de "bootstrapper" un projet RubyOnRails et de traduire lapi en MVC avec une architecture **3-tiers** (controller, model, SGBD). On va se familiariser un peu avec lenvironement ruby et la notion de gems, similaire au packages npm de `node_modules`.

1. Dans un premier temps on va brancher le projet avec une base de donnée type postgre préconfigurée sur une image docker.

2. Ensuite on soccupera de la partie **migration** pour generer la table `tickets`. La logique veut quon construise le modèle `ticket` cest a dire la classe `ticket` afin den faire un **active record**.

3. On soccupera ensuite du Routing et on mappera les routes sur un controller

4. Enfin on fera des test (mini test et pas airspec)

### Rappel notion ORM

> La notion dORM est importante (voir cours JEE et framework Hybernate). Un objet `ticket`, instance de la classe `Ticket`, est lié à un tuple de la base

```ruby
a = Ticket.new
a.id = 54321
a.ossId = 12345
a.save
```

va créer un nouveau tuple dans la base avec les valeurs correspondantes, et est exactement équivalent à la requête SQL suivante :

```sql
INSERT INTO pieces (id, ossId) VALUES (54321, 12345);
```

On parle de **ORM** (object-relational mapping)

### Initialisation dun nouveau projet

Linitialisation dun projet ruby suit un peu la logique dun projet `node.js`. On installe des gems (~ npm packages dans `node_modules`). Ces dépendances sont listées dans un fichier `Gemfile` (~ `package.json`). Pour installer ces packages on execute (~ `npm install`): `bundle install`.

> Note: changer version du projet en 2.5 et ne pas utiliser la version globale

Pour commencer un nouveau projet

```bash
rails new Projet
```

Installer le bundler

```bash
gem install bundler
```

Executer la commande bundle, lire le `Gemfile` et installer toutes les gems.

```bash
bundle install
```

### Configuration

Quelques gems utiles:

- `gem 'pg'` remplace `sqlite` par defaut cest le sgbd postgres
- `gem 'puma'` cest lequivalent dexpress pour node
- `turbolink` cest lequivalent dun router pour gerer les SPA
- `gem 'spring-watcher-listen'` cest lequivalent de `nodemon` pour faire un refresh sans restart le server

### Branchage avec base de donnee postgreSQL

Creation dune database dans le conteneur docker `pg-snapp-9.6` avec un utilisateur `snapp` et son password associé

```bash
docker run --name pg-snapp-9.6 -p 5432:5432 -e POSTGRES_PASSWORD=snapp -e POSTGRES_USER=snapp -e POSTGRES_DB=snapp -d postgres:9.6
```

Executer postgre dans le conteneur

```bash
docker exec -it pg-snapp-9.6 psql -U snapp -W -d snapp
```

Si on veut lister les containers qui ont été lancés ou qui sont lancés

```bash
docker ps -a
```

Si par exemple on a un seul container dactif et quon veut rentrer dans le bash du systeme dexploitation qui y tourne

```bash
# NR => deuxieme ligne,  $1 => 1er mot
docker ps | awk 'NR==2 {print $1}'
```

Dans le projet Rails on modifie le fichier `config/database.yml` pour la ligne developement

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000
  username: # utilisateur root par defaut
  password: # mot de passe root par defaut
  host: localhost

development:
  <<: *default
  database: fidme_ticket_dev

test:
  <<: *default
  database: fidme_ticket_test
```

Resolution du probleme dinstallation de la gem `pg`
Pour la resolution du chemin vers `pg_config`

```bash
pgpath=$(find /Applications -name pg_config)
gem install pg -- --with-pg-config=/$(echo $pgpath)
```

Resolution de linstall de `pg`

```bash
sudo su
env ARCHFLAGS="-arch x86_64" gem install pg
```

Pour installer les gems et pg sans erreur il faut finalement faire

```bash
ARCHFLAGS="-arch x86_64" bundle install
```

Creation de la base de donnee `fidme_ticket_dev`

```bash
bundle exec rake db:create
```

> remarque: les scripts dexecution rake,start... se trouvent dans le dossier `/bin`

Liste de commandes

```bash
RAILS_ENV=test bundle exec rake db:create
RAILS_ENV=test bundle exec rake db:migrate
RAILS_ENV=test bundle exec rake db:rollback
RAILS_ENV=test bundle exec rake db:drop
bundle exec rake test
```

#### Active Record Migrations

Les migrations sont des fonctionnalités de `Active Record` qui permettent de modifier le schéma de la base de donnée.
Plutot que de modifier des modifications sur le schéma en pure SQL, les migrations permettent de décrire ces changements dans un [DSL = Domain-specific language](https://en.wikipedia.org/wiki/Domain-specific_language) ruby. (altering database)

Si on prend les migration du projet `node` pour la création dun ticket

```javascript
exports.up = knex => {
  return knex.schema.createTable('tickets', table => {
    table.uuid('id').primary();
    table.dateTime('uploaded_at');
    table.dateTime('edited_at');
    table.dateTime('deleted_at').index();
    table.text('comment');
    table.string('customer_id').index();
    table.string('loyalty_card_id').index();
    table.specificType('paths', 'varchar[]');
  });
};
```

> pour info perso, remarquons que cetait knex (grunt) qui gerait les migrations

On va maintenant traduire ça en `class` ruby

```ruby
class CreateTickets < ActiveRecord::Migration[1.0]
def change
	create_table :tickets do |t|
		t.datetime :uploaded_at
		t.datetime :edited_at
		t.datetime :deleted_at
		t.text :comment
		t.integer :customer_id
		t.string :loyalty_card_id
	end
end
def down
	drop_table :tickets
end
end
```

> remarque la clé primaire est ajoutee automatiquement
> remarque si on ajoute `t.timestamps` les attributs `created_at` et `updated_at` sont aussi crées et peuplés automatiquement
