# Cahier des charges

On doit réécrire une api `node.js` en ruby on rails. Celle ci gère les tickets de caisse des clients et sinsère dans un projet plus large de reconnaissance dimage ([OCR](https://fr.wikipedia.org/wiki/Reconnaissance_optique_de_caractères) = reconnaissance optique de caracteres).

## Sommaire

1. [Objectifs](#objectifs)

2. [Model](#model)

   [Ticket/command](#ticket/command)

   [Ticket/query](#ticket/query)

3. [Controller](#controller)

   [lib/web](#ticket/web)

## Objectifs

Le pattern utilise dans le projet node est de type DDD (Domain Driven Design). Il faudra changer ce pattern en MC (model controler) sans le V car il ny a pas de vues a gerer. La structure du projet suit aussi une architecture de type [CQRS](https://docs.microsoft.com/fr-fr/azure/architecture/guide/architecture-styles/cqrs) (command query responsability segregation). Les `command` et les `query` de laggregat _Ticket_ classe de type _CRUD_ creent des _Event_ stackés dans un _Bus_. Ces _Event_ sont traités par `web` qui manipule le SGBD.

> On retrouve ici un pattern de type event/payload/applyEvent/storeSubscribe qui me fait fortement penser a Redux...

1. Pour la comprehension du projet on se concentrera dabord sur la liste des use cases quon retrouvera dans les repertoires `lib/ticket/command/handler.js` et `lib/ticjet/query/handler`
2. Ensuite on verra comment acceder a lapi depuis lexterieur dans le dossier `/lib/web`
3. Enfin dans un dernier temps on commencera un nouveau projet ruby pour re implementer lapi

> On veillera a se concentrer sur la spec et sur la definition des fonctions métiers.

## Model

### ticket/command

Pour rappel `ticket.js` est un _agregated root_ dans le language DDD. Cest un objet de type _CRUD_ (create, read, update, delete). Cette entite na pas dautre metier que de persister. Les handlers sont limplementation concrete du CRUD. `ticket.js` ne fait que suivre le pattern du DDD avec le schéma _payload_ et souscription a une liste d _events_

### Analyse

On charge les dependances du pattern _DDD_.

```
const {AggregateRoot, jsonEqual, Event} = require('ddd');
```

On type lobjet `Creation` avec `tcomb`. Cest une sorte de libraire de typage comme `flow` pour sassurer que chaque objet a le bon type et sil est optionnel (cest une chaine de caracteres `t.String()` et ce parametre est optionnel `t.maybe()`). Sa structure suit la convention _payload/messageType_

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
