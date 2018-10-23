# Cahier des charges

## Ojectif

Je dois réécrire une api `node.js` en ruby on rails. Ce projet gère les tickets de caisse des clients et sinsère dans un projet plus large de reconnaissance dimage (OCR = reconnaissance optique de caracteres). Le pattern utilise dans le projet node est de type DDD (Domain Driven Design). Il faudra changer ce pattern en MV (model controler) sans le V car il ny a pas de vues a gerer. La structure du projet suit aussi une architecture de type _CQRS_ (commabd query responsability segregation). Les commandes dun ticket envoient un _Event_ de type _CRUD_ aux _Query_

1. Pour la comprehension du projet on se concentrera dabord sur la liste des use cases quon retrouvera dans les repertoires `lib/ticket/command/handler.js` et `lib/ticjet/query/handler`
2. Ensuite on verra comment acceder a lapi depuis lexterieur dans le dossier `/lib/web`

> On veillera a se concentrer sur la spec et sur la definition des fonctions métiers.

3. Enfin dans un dernier temps on commencera un nouveau projet ruby pour re implementer lapi

## Le dossier `ticket/command`

Pour rappel `ticket.js` est un _agregated root_ dans le language DDD. Cest un _CRUD_ (create, read, update, delete). Cette entite na pas dautre metier que de persister. Les handlers sont limplementation concrete du CRUD. `ticket.js` ne fait que suivre le pattern du DDD avec le schéma _payload_ et souscription a une liste d _events_

### Analyse

On charge les dependances du pattern _DDD_.

```
const {AggregateRoot, jsonEqual, Event} = require('ddd');
```

On type lobjet `Creation` avec `tcomb`. Cest une sorte de libraire de typage comme `flow` pour sassurer que chaque objet a le bon type et sil est optionnel (cest une chaine de caracteres `t.String()` et ce parametre est optionnel `t.maybe()`)

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

On type `Update` et `CustomerDeletion`.

```javascript
const Update = t.interface(
  {
    editedAt: t.maybe(t.Date),
    comment: t.maybe(t.String),
  },
  'TicketUpdate',
);

const CustomerDeletion = t.interface(
  {
    customerId: t.String,
  },
  'CustomerDeletion',
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
- on cree un evenement `ticketCreatedEvent` avec son _payload_
- enfin on retourne lobjet ticket et sa liste devenements

```javascript
return {object: ticket, events: [ticketCreatedEvent]};
```
