# Haz Tu Propia Aplicación Web Usando React, Flux y Auth0
En este post veremos cómo utilizar tres de las tecnologías web más populares para hacer una aplicación con llamadas a una API remota y con autenticación de usuarios. Léelo!

[Accede al código de ejemplo!](https://github.com/auth0/platzi-react-flux-auth0-example)

## Introducción
El ecosistema de React es inmenso, y en él hay muchos módulos que podemos elegir para remover la fricción de las partes complejas de React. Algunas de las partes que más fricción generan son la de manejo de estado y la de ruteo. Y dos opciones populares para el manejo de estas partes son: Flux y React Router.

Nos concentraremos en este post en tratar de entender cómo utilizar estas tecnologías en un contexto real. En particular, veremos cómo utilizarlas eficientemente para realizar llamadas a una API remota, y también para autenticar usuarios.

Utilizaremos como ejemplo una simple aplicación de contactos. La misma recibirá toda su información haciendo consultas a una API remota. Del lado del servidor, utilizaremos Node.js con Express, aunque cualquier servidor puede cumplir las funciones necesarias mientras pueda manejar datos en formato JSON.

Una de las mejores formas de manejar los datos de autenticación es utilizar [JSON Web Tokens (JWTs)](https://jwt.io/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth). Desde luego, todo lo que tiene que ver con autenticación y autorización de usuarios tiende a ser complejo y sensible. Por eso, utilizaremos [Auth0](https://auth0.com/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth), un servicio de autenticación, para que se haga cargo de manejar esta complejidad por nosotros. Entre otras características, con Auth0 tendremos una pantalla de login con soporte para ingreso seguro desde Facebook, Twitter y otras plataformas sociales con solo un par de líneas de código.

![La aplicación corriendo](1.png?raw=true)

Comencemos!

## Configurando un nuevo proyecto de React
Utilizaremos React con ECMAScript/JavaScript 2015. Esto significa que necesitaremos hacer uso de un "transpiler", es decir, un compilador que se encargará de traducir nuestro JavaScript moderno a una versión anterior, a favor de que funcione correctamente en la mayor cantidad de browsers posible. Ésta es una práctica común en el mundo de JavaScript actual. Para manejar los distintos aspectos de la compilación utilizaremos [Webpack](https://webpack.github.io), un empaquetador web que, entre otras cosas, puede invocar a un compilador como parte del proceso de empaquetado. Afortunadamente, también tenemos una herramienta que podemos utilizar para ayudarnos a generar la estructura inicial del proyecto: Yeoman. Veamos:

```sh
npm install -g yo
npm install -g generator-react-webpack
mkdir react-auth && cd react-auth
yo react-webpack
```

Veremos algunas preguntas que deberemos responder. Son sencillas, manos a la obra! Una vez que las hayamos respondido, un nuevo proyecto de React debería estar disponible en el directorio que creamos.

Necesitaremos algunas dependencias adicionales. Para instalarlas:

```sh
npm install flux react-router bootstrap react-bootstrap keymirror superagent
```

También tendremos que cambiar los parámetros del módulo `url-loader` en nuestra configuración de Webpack para que React Bootstrap funcione correctamente.

```javascript
// cfg/defaults.js

// ...

{
  test: /\.(png|woff|woff2|eot|ttf|svg)$/,
  loader: 'url-loader?limit=8192'
},

// ...
```

Otra cosa que deberemos hacer es cambiar la ruta que Webpack utiliza para servir el proyecto, de lo contrario tendremos problemas con React Router. Para hacer esto, abriremos `server.js` y hacia el fin del mismo cambiaremos lo siguiente:

```javascript
// Cambiar esto:
open('http://localhost:' + config.port + '/webpack-dev-server/');
// Por esto:
open('http://localhost:' + config.port);
```

## Express como servidor
Para la parte del servidor, crearemos una aplicación basada en Express. Por suerte, esto es algo relativamente sencillo. Primero crearemos un directorio aparte con algunas dependencias:

```sh
mkdir react-auth-server && cd react-auth-server
npm init
npm install express express-jwt cors
touch server.js
```

El paquete `express-jwt` se encargará de ayudarnos con la autenticación del lado del servidor. Express utiliza el concepto de `middlewares` para establecer pasos adicionales a realizar durante el acceso a una URL determinada de nuestra aplicación. `express-jwt` es un middleware.

Agregaremos lo siguiente a `server.js`:

```javascript
// server.js

const express = require('express');
const app = express();
const jwt = require('express-jwt');
const cors = require('cors');

app.use(cors());

// Este es el middleware de express-jwt configurado utilizando los parámetros
// de su cuenta de Auth0
const authCheck = jwt({
  secret: new Buffer('YOUR_AUTH0_SECRET', 'base64'),
  audience: 'YOUR_AUTH0_CLIENT_ID'
});

var contacts = [
  {
    id: 1,
    name: 'Kim',
    email: 'kim@email.com',
    image: 'https://en.gravatar.com/userimage/20807150/4c9e5bd34750ec1dcedd71cb40b4a9ba.png'
  },
  {
    id: 2,
    name: 'Gonto',
    email: 'gonto@email.com',
    image: 'https://www.gravatar.com/avatar/df6c864847fba9687d962cb80b482764??s=200'
  },
  {
    id: 3,
    name: 'Ado',
    email: 'ado@email.com',
    image: '//gravatar.com/avatar/99c4080f412ccf46b9b564db7f482907?s=200'
  },
  {
    id: 4,
    name: 'Sebastián',
    email: 'sebastian@email.com',
    image: 'http://en.gravatar.com/userimage/92476393/001c9ddc5ceb9829b6aaf24f5d28502a.png?size=200'
  },
  {
    id: 5,
    name: 'Ryan',
    email: 'ryan@email.com',
    image: '//gravatar.com/avatar/7f4ec37467f2f7db6fffc7b4d2cc8dc2?s=200'
  }
];

app.get('/api/contacts', (req, res) => {
  const allContacts = contacts.map(contact => { 
    return { id: contact.id, name: contact.name}
  });
  res.json(allContacts);
});

app.get('/api/contacts/:id', authCheck, (req, res) => {
  res.json(contacts.filter(contact => contact.id === parseInt(req.params.id))[0]);
});

app.listen(3001);
console.log('Listening on http://localhost:3001');
```

En este bloque de código tenemos una lista de contactos que es retornada de dos URLs distintas. Para la URL `/api/contacts` utilizamos `map` para retornar sólo los campos `id` y `name`. Para la URL `/api/contacts/:id` utilizamos el `id` pasado como parámetro para retornar todos los datos asociados a dicho `id`. Esta URL en particular requiere de autenticación. Para mantener el ejemplo lo más sencillo posible, en lugar de hacer uso de una base de datos, la lista de contactos se encuentra directamente integrada en el código.

## Auth0 como servidor de autenticación
En el paso anterior utilizamos la función `authCheck` como middleware para autenticar el acceso a una URL. Afortunadamente el funcionamiento de esto es relativamente sencillo. Al acceder dicha URL, un JSON Web Token (JWT) debe estar incluido en el pedido HTTP. Para validar dicho JWT, el middleware debe conocer dos datos: el id de cliente de Auth0, y la clave secreta correspondiente. Estos datos los podemos obtener de nuestra cuenta de Auth0.

Si aún no te has suscripto a Auth0, éste es el momento de hacerlo. Entra a [auth0.com](https://auth0.com/signup/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth) y sigue los pasos. Una vez que te hayas registrado, encontrarás los datos de tu aplicación en el área de [administración](https://manage.auth0.com/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth). Busca `Client ID` y `Client secret`. Una vez que tengas esos datos, complétalos en `server.js`.

Asimismo, en el apartado de `Allowed Origins` (orígenes permitidos) debe completarse `https://localhost:8000`, para permitir que este ejemplo pueda correrse de manera local.

![Configuración de Auth0](2.png?raw=true)

## Configurando el índice y el ruteo
Modificaremos el archivo `index.js`.

```javascript
// src/index.js

import 'core-js/fn/object/assign';
import React from 'react';
import ReactDOM from 'react-dom';
import { browserHistory } from 'react-router';
import Root from './Root';

// Dibujando el componente principal en el DOM
ReactDOM.render(<Root history={browserHistory} />, document.getElementById('app'));
```

En este paso estamos dibujando el contenido de un componente llamado `Root` al cual le pasamos una propiedad llamada `browserHistory` dentro de un elemento preexistente en el DOM llamado `app`.

Para finalizar con la parte de ruteo, deberemos crear un archivo llamado `Root.js`. Este archivo se encargará de establecer nuestras rutas.

```javascript
// Root.js

import React, { Component } from 'react';
import { Router, Route, IndexRoute } from 'react-router';
import Index from './components/Index';
import ContactDetail from './components/ContactDetail';

import App from './components/App';

class Root extends Component {

  // Es necesario proveer una lista de rutas para nuestra aplicación,
  // y, en este caso, lo hacemos desde el componente Root.
  render() {
    return (
      <Router history={this.props.history}>
        <Route path='/' component={App}>
          <IndexRoute component={Index}/>
          <Route path='/contact/:id' component={ContactDetail} />
        </Route>
      </Router>
    );
  }
}

export default Root;
```

Con React Router podemos ubicar rutas individuales dentro una ruta principal (indicada con el elemento `Router`). El elemento `Router` utiliza un parámetro llamado `history` que es a su vez utilizado para leer la URL principal de la barra de direcciones y construir objetos que representan ubicaciones específicas. El objecto `history` es enviado desde el código presente en `index.js`, que creamos más arriba.

Ésta es también una buena oportunidad para incluir el componente `Lock` en nuestra aplicación. `Lock` es el componente de Auth0 que se encarga de mostrar una pantalla de login. Este componente puede ser instalado desde `npm`, o puede ser incluido directamente como un tag `<script>` en el código HTML. Para mantener las cosas simples, lo agregaremos en el HTML:

```html
<!-- src/index.html --> 
...

<!-- Auth0Lock script -->
<script src="//cdn.auth0.com/js/lock-9.1.min.js"></script>

<!-- Setting the right viewport -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />   

...
```

## El componente App
El primer componente que deberíamos configurar a partir de este punto debería ser el que representa a la aplicación. Llamaremos a este componente `App`. Para mantener las cosas claras, renombraremos `Main.js` a `App.js` e incluiremos en él algunos de los componentes de React Bootstrap.

```javascript
// src/components/App.js

import 'normalize.css/normalize.css';
import 'bootstrap/dist/css/bootstrap.min.css';

import React, { Component } from 'react';
import Header from './Header';
import Sidebar from './Sidebar';
import { Grid, Row, Col } from 'react-bootstrap';

class AppComponent extends Component {

  componentWillMount() {
    this.lock = new Auth0Lock('YOUR_AUTH0_CLIENT_ID', 'YOUR_AUTH0_DOMAIN');
  }

  render() {
    return (
      <div>
        <Header lock={this.lock}></Header>
        <Grid>
          <Row>
            <Col xs={12} md={3}>
              <Sidebar />
            </Col>
            <Col xs={12} md={9}>
              {this.props.children}
            </Col>
          </Row>
        </Grid>
      </div>
    );
  }
}

export default AppComponent;
```

Encontraremos en el código referencias a otros componentes: `Header` y `Sidebar`. Crearemos esos componentes a continuación, pero por el momento nos concentraremos en lo que ocurre dentro de `componentWillMount`. Éste es el lugar donde configuraremos nuestra instancia de `Auth0Lock`. Esto es tan sencillo como crear una nueva instancia de dicho componente: `new Auth0Lock` pasando nuestro `Client ID` y `Client Domain` (ambos disponibles en el panel de administración de Auth0). 

Un detalle interesante para notar es el uso de `this.props.children` en el segundo componente `Col`. Esto permite mostrar rutas hijas de React Router dentro de este elemento. De esta manera, podemos tener una barra lateral fija con una vista dinámica en la parte central de nuestra aplicación.

A su vez, estamos pasando la instancia de `Auth0Lock` como una propiedad a `Header`. Veamos este componente a continuación.

## El componente Header
En este paso pondremos una barra de navegación que también servirá como ubicación para los botones que permitirán a los usuarios ingresar al sistema y salir de él.

```javascript
// src/components/Header.js

import React, { Component } from 'react';
import { Nav, Navbar, NavItem, Header, Brand } from 'react-bootstrap';
// import AuthActions from '../actions/AuthActions';
// import AuthStore from '../stores/AuthStore';

class HeaderComponent extends Component {

  constructor() {
    super();
    this.state = {
      authenticated: false
    }
    this.login = this.login.bind(this);
    this.logout = this.logout.bind(this);
  }

  login() {
    // Podemos llamar al método show de Auth0Lock, que es pasado como una
    // propiedad, para permitir que el usuario se autentique
    this.props.lock.show((err, profile, token) => {
      if (err) {
        alert(err);
        return;
      }
      this.setState({authenticated: true});
    });
  }

  logout() {
    // AuthActions.logUserOut();
    this.setState({authenticated: false});
  }

  render() {
    return (
      <Navbar>
        <Navbar.Header>
          <Navbar.Brand>
            <a href="#">React Contacts</a>
          </Navbar.Brand>
        </Navbar.Header>
        <Nav>
          <NavItem onClick={this.login}>Login</NavItem>
          <NavItem onClick={this.logout}>Logout</NavItem>
        </Nav>
      </Navbar>
    );
  }
}

export default HeaderComponent;
```

A esta altura las cosas aún no son del todo claras. Nos faltan definir acciones y lugares donde guardar el estado de la aplicación, que aclararán algunas cosas. El método `login` es el que produce que la pantalla de login sea mostrada. El elemento `NavItem` con el texto `Login` produce la llamada a este método al ser cliqueado. Por el momento solo estamos modificando el valor de la variable `authenticated`. Más adelante veremos cómo asociar el valor de esta variable a la presencia de un JWT válido. 

Antes de que podamos ver algo en la pantalla, aún nos quedan por configurar los componentes `Sidebar` e `Index`.

## Los componentes Sidebar e Index
```javascript
// src/components/Sidebar.js

import React, { Component } from 'react';

class SidebarComponent extends Component {
  render() {
    return (
      <h1>Hello from the sidebar</h1>
    );
  }
}

export default SidebarComponent;
```

Ésta es el lugar donde eventualmente se encontrará la lista de contactos, pero por el momento sólo mostraremos un mensaje.

También tenemos un componente `Index` que es utilizado por el ruteador como `IndexRoute`. Este componente mostrará un mensaje acerca de hacer clic en un contacto para ver su perfil.

```javascript
// src/components/Index.js

import React, { Component } from 'react';

class IndexComponent extends Component {

  constructor() {
    super();
  }
  render() {
    return (
      <h2>Click on a contact to view their profile</h2>
    );
  }
}

export default IndexComponent;
```

Finalmente estamos listos para ver nuestra aplicación por primera vez! Pero primero, es necesario que comentemos algunas cosas que se encuentran en `Root.js`. Aún no tenemos un componente `ContactDetail`, así que por el momento removeremos su `import` y su ruta asociada.

Si todo sale bien, así debería verse la aplicación a esta altura:

![Vista inicial de la aplicación](3.png?raw=true)

También deberíamos poder ver la pantalla de autenticación al hacer clic en `login`.

![Auth0 Lock](4.png?raw=true)

## Implementando Flux
Flux es una gran herramienta para el manejo de estado, pero una de sus desventajas es que requiere bastante código. Para mantener esta sección lo más corta posible, evitaremos entrar en detalles de Flux. Hay muy buenos tutoriales y explicaciones en la web.

Para ser lo más concretos posible, Flux es una arquitectura que nos permite manejar el flujo de datos de nuestras aplicaciones. Este flujo es, por diseño, unidireccional. Una arquitectura donde el flujo de datos va en una única dirección es importante en aplicaciones de tamaño considerable. Esto se debe a que es más sencillo razonar en aplicaciones de estas características.

Los conceptos fundamentales de Flux son las **acciones** (actions), el **despachante** (dispatcher), y los **almacenes de datos** (stores).

### El despachante
Comencemos por crear el despachante.

```javascript
// src/dispatcher/AppDispatcher.js

import { Dispatcher } from 'flux';

const AppDispatcher = new Dispatcher();

export default AppDispatcher;
```

Solamente debe haber un único despachante por aplicación de React + Flux, éste es creado mediante la llamada a `new Dispatcher()`.

### Las acciones
A continuación crearemos las acciones que nos permitirán bajar los datos de nuestra lista de contactos a través de la API.

```javascript
// src/actions/ContactActions.js

import AppDispatcher from '../dispatcher/AppDispatcher';
import ContactConstants from '../constants/ContactConstants';
import ContactsAPI from '../utils/ContactsAPI';

export default {

  recieveContacts: () => {
    ContactsAPI
      .getContacts('http://localhost:3001/api/contacts')
      .then(contacts => {
        AppDispatcher.dispatch({
          actionType: ContactConstants.RECIEVE_CONTACTS,
          contacts: contacts
        });
      })
      .catch(message => {
        AppDispatcher.dispatch({
          actionType: ContactConstants.RECIEVE_CONTACTS_ERROR,
          message: message
        });
      });
  },

  getContact: (id) => {
    ContactsAPI
      .getContact('http://localhost:3001/api/contacts/' + id)
      .then(contact => {
        AppDispatcher.dispatch({
          actionType: ContactConstants.RECIEVE_CONTACT,
          contact: contact
        });
      })
      .catch(message => {
        AppDispatcher.dispatch({
          actionType: ContactConstants.RECIEVE_CONTACT_ERROR,
          message: message
        });
      });
  }

}
```

En Flux las acciones deben despachar un `actionType` (alguno de los tipos predefinidos de acciones posibles) y datos arbitrarios (un mensaje). Normalmente, estos datos se encuentran asociados con la acción y son obtenidos a través del almacén de datos conectado a la misma.

Hay opiniones opuestas con respecto a si realizar llamadas a URLs de la API dentro de acciones es una buena idea. Algunos creen que este tipo de actividades deben ser realizadas por los almacenes de datos. En última instancia, el lugar que se elija para realizar este tipo de actividades queda a criterio del desarrollador y lo que sea mejor para la aplicación en cuestión.

En el bloque de código precedente se encuentran `ContactsAPI` y `ContactConstants`, dos elementos que aún no hemos definido.

### Las constantes de contactos
```javascript
// src/constants/ContactConstants.js

import keyMirror from 'keymirror';

export default keyMirror({
  RECIEVE_CONTACT: null,
  RECIEVE_CONTACTS: null,
  RECIEVE_CONTACT_ERROR: null,
  RECIEVE_CONTACTS_ERROR: null
});
```

Estas constantes nos permiten diferenciar entre distintos tipos de acciones y en el futuro las utilizaremos como nexo entre las acciones y los almacenes de datos. Utilizamos la librería `keyMirror` para que los valores del objeto que creamos implícitamente tenga claves y valores iguales.

### La API de contactos
A esta altura tenemos una noción de como nuestra API de contactos (`ContactsAPI`) debería verse desde el punto de vista de las acciones. Queremos exponer funciones para enviar pedidos `XmlHttpRequest` al servidor, retornando promesas de JavaScript desde los mismos. Para los pedidos XHR utilizaremos la librería `superagent` que provee una API muy sencilla y conveniente.

```javascript
// src/utils/ContactsAPI.js

import request from 'superagent/lib/client';

export default {

  // Queremos obtener una lista de todos los contactos a través de la API.
  // Esta lista contiene información reducida y será utilizada para la lista
  // de la izquierda en la pantalla.
  getContacts: (url) => {
    return new Promise((resolve, reject) => {
      request
        .get(url)
        .end((err, response) => {
          if (err) reject(err);
          resolve(JSON.parse(response.text));
        })
    });
  },

  getContact: (url) => {
    return new Promise((resolve, reject) => {
      request
        .get(url)
        .end((err, response) => {
          if (err) reject(err);
          resolve(JSON.parse(response.text));
        })
    });
  }
}
```

Con la librería `superagent` llamamos al método `get` para realizar un pedido HTTP `GET`. En el método `end` establecemos el callback que procesará el resultado o el error correspondiente.

Si encontramos algún error en los pedidos XHR, utilizamos el método `reject` de la promesa directamente. El método `catch` capturará el error en la acción correspondiente. En cambio, si el pedido tiene éxito llamamos al método `resolve` con los datos correspondientes.

### El almacén de datos de contactos
Antes de poder dibujar algo en la pantalla, es necesario que creemos el almacén de datos de contactos. 

```javascript
// src/stores/ContactStore.js

import AppDispatcher from '../dispatcher/AppDispatcher';
import ContactConstants from '../constants/ContactConstants';
import { EventEmitter } from 'events';

const CHANGE_EVENT = 'change';

let _contacts = [];
let _contact = {};

function setContacts(contacts) {
  _contacts = contacts;
}

function setContact(contact) {
  _contact = contact;
}

class ContactStoreClass extends EventEmitter {

  emitChange() {
    this.emit(CHANGE_EVENT);
  }

  addChangeListener(callback) {
    this.on(CHANGE_EVENT, callback)
  }

  removeChangeListener(callback) {
    this.removeListener(CHANGE_EVENT, callback)
  }

  getContacts() {
    return _contacts;
  }

  getContact() {
    return _contact;
  }

}

const ContactStore = new ContactStoreClass();

// Aquí registramos un callback para el despachante. Cada tipo de acción
// es manejado por separado.
ContactStore.dispatchToken = AppDispatcher.register(action => {

  switch(action.actionType) {
    case ContactConstants.RECIEVE_CONTACTS:
      setContacts(action.contacts);
      // Es necesario llamar a emitChange para que el listener de eventos
      // registre que un cambio ha sido realizado
      ContactStore.emitChange();
      break

    case ContactConstants.RECIEVE_CONTACT:
      setContact(action.contact);
      ContactStore.emitChange();
      break

    case ContactConstants.RECIEVE_CONTACT_ERROR:
      alert(action.message);
      ContactStore.emitChange();
      break

    case ContactConstants.RECIEVE_CONTACTS_ERROR:
      alert(action.message);
      ContactStore.emitChange();
      break

    default:
  }

});

export default ContactStore;
```

Para los almacenes es usual utilizar un `switch` asociado a `AppDispatcher` para responder a las acciones de la aplicación. La acción `RECEIVE_CONTACTS` indica que los datos de los contactos están siendo recibidos. En este caso, los guardamos en forma de array en la función `setContacts`. Luego le indicamos a `EventListener` que emita una señal de cambio para que el resto de la aplicación pueda realizar las acciones deseadas.

La lógica de la aplicación permite recibir los datos de todos los contactos, o de uno solo (con mayor cantidad de detalles).

Antes de que podamos ver los contactos, aún nos quedan algunos componentes por crear.

## El componente de contactos
El componente `Contacts` será utilizado en la barra lateral izquierda. Utilizaremos a la vez un elemento `Link` para mostrar más detalles.

```javascript
// src/components/Contacts.js

import React, { Component } from 'react';
import { ListGroup } from 'react-bootstrap';
// import { Link } from 'react-router';
import ContactActions from '../actions/ContactActions';
import ContactStore from '../stores/ContactStore';
import ContactListItem from './ContactListItem';

// Utilizaremos esta función para crear un elemento individual de la lista
// para cada contacto.
function getContactListItem(contact) {
  return (
    <ContactListItem
      key={contact.id}
      contact={contact}
    />
  );
}
class ContactsComponent extends Component {

  constructor() {
    super();
    // Para el estado inicial de la aplicación sólo queremos un array
    // de contactos vacío.
    this.state = {
      contacts: []
    }
    // Es necesario que utilicemos bind para tener la referencia this correcta 
    // dentro de onChange.
    this.onChange = this.onChange.bind(this);
  }

  componentWillMount() {
    ContactStore.addChangeListener(this.onChange);
  }

  componentDidMount() {
    ContactActions.recieveContacts();
  }

  componentWillUnmount() {
    ContactStore.removeChangeListener(this.onChange);
  }

  onChange() {
    this.setState({
      contacts: ContactStore.getContacts()
    });
  }

  render() {
    let contactListItems;
    if (this.state.contacts) {
      // Para cada contacto, crear un elemento correspondiente.
      contactListItems = this.state.contacts.map(contact => getContactListItem(contact));
    }
    return (
      <div>
        <ListGroup>
          {contactListItems}
        </ListGroup>
      </div>
    );
  }
}

export default ContactsComponent;
```

Empezamos con un estado inicial en el constructor (`this.state`). Es crucial usar `bind` para establecer el valor de `this` en el miembro `onChange` (ver constructor).

Cuando el componente es montado por React, pedimos la lista de contactos mediante `ContactActions.receiveContacts`. Este método envía un pedido XHR al servidor (ver `ContactsAPI`) que disparará el método correspondiente en `ContactStore`. Es necesario agregar un listener en `componentWillMount` que utilice `onChange`. Este método se encargará de establecer el estado del componente de React con la lista de contactos.

Cuando esto ocurre, la lista de contactos es convertida en un conjunto de items para mostrar en pantalla (dentro de `ListGroup`, un componente de React Bootstrap). Aún nos queda por ver `ContactListItem`.

### El elemento de contacto individual
El elemento `ContactListItem` debe crear un elemento `ListGroupItem` (otro componente de React Bootstrap) con un subelemento de React Router: `Link`. Este elemento nos llevará a los detalles del contacto.

```javascript
// src/components/ContactListItem.js

import React, { Component } from 'react';
import { ListGroupItem } from 'react-bootstrap';
import { Link } from 'react-router';

class ContactListItem extends Component {
  render() {
    const { contact } = this.props;
    return (
      <ListGroupItem>
        <Link to={`/contact/${contact.id}`}>          
          <h4>{contact.name}</h4>
        </Link>
      </ListGroupItem>
    );
  }
}

export default ContactListItem;
```

En este caso simplemente recibimos el contacto como parte de `prop` y luego mostramos su nombre (propiedad `name`).

### La barra lateral
El último ajuste que debemos realizar para poder visualizar la lista de contactos es reemplazar el mensaje provisorio que habíamos utilizado anteriormente.

```javascript
// src/components/Sidebar.js

import React, { Component } from 'react';
import Contacts from './Contacts';

class SidebarComponent extends Component {
  render() {
    return (
      <Contacts />
    );
  }
}

export default SidebarComponent;
```

Con este último cambio, deberíamos poder visualizar nuestra lista de contactos.

![Lista de Contactos](5.png?raw=true)

## El componente de detalle de contacto
Una de las últimas piezas que nos queda por realizar en nuestra aplicación es el detalle de contactos. Cuando un contacto es seleccionado, los detalles del mismo deben mostrarse en la porción central de la aplicación. Los detalles son obtenidos mediante una petición XHR.

Si revisamos la configuración del servidor con Express que realizamos con anterioridad, veremos que la URL correspondiente a los detalles de contacto se encuentra protegida con autenticación (ver `authCheck` de `server.js`). En otras palabras, un JWT válido es necesario para acceder a estos datos.

Tanto las acciones como el almacén de datos ya están listos para soportar los detalles de un contacto, lo que nos falta es el componente correspondiente.

```javascript
// src/components/ContactDetail.js

import React, { Component } from 'react';
import ContactActions from '../actions/ContactActions';
import ContactStore from '../stores/ContactStore';

class ContactDetailComponent extends Component {

  constructor() {
    super();
    this.state = {
      contact: {}
    }
    this.onChange = this.onChange.bind(this);
  }

  componentWillMount() {
    ContactStore.addChangeListener(this.onChange);
  }

  componentDidMount() {
    ContactActions.getContact(this.props.params.id);
  }

  componentWillUnmount() {
    ContactStore.removeChangeListener(this.onChange);
  }

  componentWillReceiveProps(nextProps) {
    this.setState({
      contact: ContactActions.getContact(nextProps.params.id)
    });
  }

  onChange() {
    this.setState({
      contact: ContactStore.getContact(this.props.params.id)
    });
  }

  render() {
    let contact;
    if (this.state.contact) {
      contact = this.state.contact;
    }
    return (
      <div>
        { this.state.contact &&
          <div>
            <img src={contact.image} width="150" />
            <h1>{contact.name}</h1>
            <h3>{contact.email}</h3>
          </div>
        }
      </div>
    );
  }
}

export default ContactDetailComponent;
```

Este componente es similar a nuestro componente `Contacts`. La diferencia principal es que opera sobre un único contacto. El método `getContact` de `ContactActions` y `ContactStore` recibe un `id`. Este parámetro `id` viene directamente de React Router y es parte de `params`. El método `componentWillReceiveProps` es utilizado para obtener `id` de `params` al seleccionar un contacto.

### Configurando la ruta de detalle de contacto
Antes de que podamos hacer algo con todo esto, debemos configurar la ruta `ContactDetail` en `Root.js`.

```javascript
// src/Root.js

...

render() {
    return (
      <Router history={this.props.history}>
        <Route path='/' component={App}>
          <IndexRoute component={Index}/>
          <Route path='/contact/:id' component={ContactDetail} />
        </Route>
      </Router>
    );
}

...
```

A esta altura ya es posible hacer click en un contacto. Sin embargo aún no podemos acceder a los detalles.

![Error de Acceso](6.png?raw=true)

El motivo de este error de autorización es que tenemos el `middleware` que protege el acceso a la URL de detalles de contactos. Es necesario que el servidor reciba de parte del cliente un JWT válido. Para eso, el cliente primero debe ser autenticado. Nos concentraremos en eso a continuación.

## Autenticación
¿Qué es lo que ocurre exactamente cuándo un usuario realiza el proceso de login a través de Auth0? Cuando un usuario realiza el proceso de login completo, una serie de elementos son retornados. El más interesante de estos es `id_token`, un JWT que contiene información acerca del usuario autenticado. Otros elementos retornados son un [access token](https://auth0.com/docs/tokens/access_token/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth) y un [refresh token](https://auth0.com/docs/refresh-token/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth).

Las buenas noticias es que al tener este elemento en el cliente ya tenemos una buena parte del proceso de autenticación para nuestra aplicación realizado. Veremos como utilizar Flux para manejar los datos retornados por el proceso de autenticación.

### Acciones de autenticación
```javascript
// src/actions/AuthActions.js

import AppDispatcher from '../dispatcher/AppDispatcher';
import AuthConstants from '../constants/AuthConstants';

export default {

  logUserIn: (profile, token) => {
    AppDispatcher.dispatch({
      actionType: AuthConstants.LOGIN_USER,
      profile: profile,
      token: token
    });
  },

  logUserOut: () => {
    AppDispatcher.dispatch({
      actionType: AuthConstants.LOGOUT_USER
    });
  }

}
```

Esta parte es parecida a lo que realizamos para `ContactActions`, con la excepción de que estamos concentrados en las acciones de `login` y `logout`. En el método `logUserIn` despachamos como datos `profile` (el perfil del usuario) y `token` (el JWT) que vendrán de nuestro componente `Header`.

### Constantes de autenticación
Necesitamos algunas constantes nuevas para la autenticación.

```javascript
// src/constants/AuthConstants.js

import keyMirror from 'keymirror';

export default keyMirror({
  LOGIN_USER: null,
  LOGOUT_USER: null
});
```

### El almacén de datos de autenticación
El objeto `AuthStore` es el que se encargará finalmente de recibir los datos de autenticación (el perfil y el JWT). Lo más conveniente para realizar con estos elementos es simplemente guardarlos en `LocalStorage` para poder acceder a ellos cuando haga falta en el futuro (dando persistencia a la aplicación).

```javascript
// src/stores/AuthStore.js

import AppDispatcher from '../dispatcher/AppDispatcher';
import AuthConstants from '../constants/AuthConstants';
import { EventEmitter } from 'events';

const CHANGE_EVENT = 'change';

function setUser(profile, token) {
  if (!localStorage.getItem('id_token')) {
    localStorage.setItem('profile', JSON.stringify(profile));
    localStorage.setItem('id_token', token);
  }
}

function removeUser() {
  localStorage.removeItem('profile');
  localStorage.removeItem('id_token');
}

class AuthStoreClass extends EventEmitter {
  emitChange() {
    this.emit(CHANGE_EVENT);
  }

  addChangeListener(callback) {
    this.on(CHANGE_EVENT, callback)
  }

  removeChangeListener(callback) {
    this.removeListener(CHANGE_EVENT, callback)
  }

  isAuthenticated() {
    if (localStorage.getItem('id_token')) {
      return true;
    }
    return false;
  }

  getUser() {
    return localStorage.getItem('profile');
  }

  getJwt() {
    return localStorage.getItem('id_token');
  }
}

const AuthStore = new AuthStoreClass();

// Aquí registramos un callback para el despachante y establecemos respuestas
// para cada tipo de acción.
AuthStore.dispatchToken = AppDispatcher.register(action => {

  switch(action.actionType) {

    case AuthConstants.LOGIN_USER:
      setUser(action.profile, action.token);
      AuthStore.emitChange();
      break

    case AuthConstants.LOGOUT_USER:
      removeUser();
      AuthStore.emitChange();
      break

    default:
  }

});

export default AuthStore;
```

La función `setUser` es llamada cuando un login es realizado con éxito. En ese caso, los datos son guardados en `LocalStorage`. También hemos escrito algunos métodos de utilidad que nos ayudarán con los componentes. Uno de ellos es `isAuthenticated`, que nos permitirá mostrar u ocultar elementos de manera condicional (de acuerdo al estado de autenticación del usuario).

En una aplicación tradicional, cuando el usuario realiza un login exitoso una sesión es establecida por el servidor, y es esta sesión la que luego es utilizada para establecer si el usuario se encuentra autenticado o no. Sin embargo, cuando se trata de autenticación con JWTs, no hay estado. El servidor simplemente valida el JWT recibido (mediante un secreto compartido o un certificado digital). Ni la sesión ni el estado son necesarios. Hay algunos motivos por los cuales esto puede ser algo positivo. Sin embargo, esto no nos quita la duda de cómo chequear si un usuario se encuentra autenticado o no.

Las buena noticia es que esto es más sencillo de lo que parece: sólo debemos verificar que haya un token en `LocalStorage`. Si es así, podemos asumir que el usuario se encuentra autenticado. Si el mismo es inválido, cualquier operación que el usuario realice para acceder al servidor será denegada, resultando en que el mismo deba realizar nuevamente el proceso de login. Es posible realizar una verificación adicional del lado del cliente: chequear que el tiempo de expiración del token no haya sido excedido. Sin embargo esto no es necesario y para nuestro ejemplo dejar que el token sea rechazado en caso de expiración cumple perfectamente nuestras necesidades.

### Adaptando el componente Header para autenticación
Nos quedaría adaptar el componente `Header` para que haga uso de `AuthActions` y `AuthStore` para despachar las acciones.

```javascript
// src/components/Header.js

...

import AuthActions from '../actions/AuthActions';
import AuthStore from '../stores/AuthStore';

class HeaderComponent extends Component {

  // ...

  login() {
    this.props.lock.show((err, profile, token) => {
      if (err) {
        alert(err);
        return;
      }
      AuthActions.logUserIn(profile, token);
      this.setState({authenticated: true});
    });
  }

  logout() {
    AuthActions.logUserOut();
    this.setState({authenticated: false});
  }

  // ...

}
```

Con estos cambios realizados, podemos realizar el proceso de login y ver cómo el perfil y el JWT del usuario son guardados en `LocalStorage`.

![Datos en LocalStorage](7.png?raw=true)

## Realizando una petición XHR autenticada
Cada petición que el cliente realice al servidor y que requiera autenticación debe ser realizada con el JWT obtenido en el login incluido en la misma. El JWT es usualmente pasado como parte de la cabecera HTTP de la petición, bajo la clave `Authorization`. Usando la librería `superagent` esto es muy sencillo.

```javascript
// src/utils/ContactsAPI.js

import AuthStore from '../stores/AuthStore';

// ...

  getContact: (url) => {
    return new Promise((resolve, reject) => {
      request
        .get(url)
        .set('Authorization', 'Bearer ' + AuthStore.getJwt())
        .end((err, response) => {
          if (err) reject(err);
          resolve(JSON.parse(response.text));
        })
    });
  }

// ...

```

Configuramos la cabecera `Authorization` con el prefijo `Bearer` seguido de nuestro JWT (el cual podemos obtener de nuestro almacén). Con esto presente, ya deberíamos poder acceder a los detalles de nuestros contactos.

![Los detalles de contacto](1.png?raw=true)

## Toques finales: muestra condicional de elementos
Ya casi tenemos todo en nuestro ejemplo. Nos quedaría poder mostrar condicionalmente algunos elementos. El elemento `Login` de la barra superior lo queremos mostrar únicamente si el usuario no se encuentra autenticado. Lo opuesto ocurre para el elemento `Logout`.

```javascript
// src/components/Header.js

// ...

  constructor() {
    super();
    this.state = {
      authenticated: AuthStore.isAuthenticated()
    }
    // ...
  }

  // ...

  render() {
    return (
      <Navbar>
        <Navbar.Header>
          <Navbar.Brand>
            <a href="#">React Contacts</a>
          </Navbar.Brand>
        </Navbar.Header>
        <Nav>
          { !this.state.authenticated ? (
            <NavItem onClick={this.login}>Login</NavItem>
          ) : (
            <NavItem onClick={this.logout}>Logout</NavItem>
          )}
        </Nav>
      </Navbar>
    );
  }

// ...
```

En este caso estamos utilizando el estado de autenticación del usuario directamente del almacén mientras carga el componente. Este estado es utilizado para mostrar y ocultar los elementos `NavItem` condicionalmente.

Algo similar podemos hacer para el mensaje en nuestro componente `Index`.

```javascript

// ...

  constructor() {
    super();
    this.state = {
      authenticated: AuthStore.isAuthenticated()
    }
  }
  render() {
    return (
      <div>
        { !this.state.authenticated ? (
          <h2>Log in to view contact details</h2>
        ) : (
          <h2>Click on a contact to view their profile</h2>
        )}
      </div>
    );
  }

// ...
```

## Conclusión
Si has leído hasta el final, tienes una aplicación de React + Flux que llama a una API remota y realiza autenticación a través de Auth0. Buen trabajo!

No hay duda al respecto: crear una aplicación de React + Flux puede requerir bastante código. Para pequeños proyectos puede ser difícil visualizar los beneficios de esta manera de hacer las cosas. Sin embargo, el flujo de datos unidireccional y la estructura de aplicaciones a la que conlleva el uso de Flux se vuelve algo cada vez más importante a medida que la aplicación crece. En particular, evitar las complicaciones del flujo de datos bidireccional es esencial para facilitar el razonamiento acerca del funcionamiento de la aplicación.

Afortunadamente, la parte de autenticación, que algunas veces puede ser difícil, queda en manos de Auth0. Si el backend es realizado con una tecnología diferente a Node.js, Auth0 soporta los SDKs más populares. Algunos de ellos (¡pero no todos!) son:

- [Laravel](https://auth0.com/docs/quickstart/backend/php-laravel/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth)
- [Go](https://auth0.com/docs/quickstart/backend/golang/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth)
- [Ruby on Rails](https://auth0.com/docs/quickstart/backend/rails/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth)
- [Firebase](https://auth0.com/docs/quickstart/backend/firebase/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth)
- [Python](https://auth0.com/docs/quickstart/backend/python/?utm_source=platzi&utm_medium=gp&utm_campaign=react_auth)
