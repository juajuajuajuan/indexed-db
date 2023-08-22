# TODO list

App de prueba para probar cómo funcionan las indexed-db en front-end. Se trata de una base de datos NoSQL.

## Arquitectura

Hay un HTML que contiene el esqueleto, un estilo CSS muy básico y el JS que inicia, agrega y elimina registros de la IndexedDB.

## Qué es IndexedDB

Una API de bajo nivel para almacenar en front-end. Puedes guardar ficheros, blobs, imágenes, vídeos o datos estructurados (JSON, objetos, listas, arrays)

## Info

Contiene una Base de Datos llamada `todo_db`, en versión 1, y una tabla/object store llamada `todo_tb`. A esa "tabla" se le agregan dos columnas, `title` y `desc`.

```javascript
let db;
const openOrCreateDB = window.indexedDB.open('todo_db', 1);

openOrCreateDB.addEventListener('error', () => console.error('Error opening DB'));

openOrCreateDB.addEventListener('success', () => {
  console.log('Successfully opened DB');
  db = openOrCreateDB.result;
  showTodos();
});

openOrCreateDB.addEventListener('upgradeneeded', init => {
  db = init.target.result;

  db.onerror = () => {
    console.error('Error loading database.');
  };

  const table = db.createObjectStore('todo_tb', { keyPath: 'id', autoIncrement:true });

  table.createIndex('title', 'title', { unique: false });
  table.createIndex('desc', 'desc', { unique: false });
});
```


## Agregar registros

Para agregar un registro, se lee del formulario y se llama a guardar en la transacción. Al guardar, limpia el formulario y recarga los todos, para volver a mostrarlos.

```javascript
form.addEventListener('submit', addTodo);

function addTodo(e) {
  e.preventDefault();
  const newTodo = { title: todoTitle.value, body: todoDesc.value };
  const transaction = db.transaction(['todo_tb'], 'readwrite');
  const objectStore = transaction.objectStore('todo_tb');
  const query = objectStore.add(newTodo);
  query.addEventListener('success', () => {
    todoTitle.value = '';
    todoDesc.value = '';
  });
  transaction.addEventListener('complete', () => {
    showTodos();
  });
  transaction.addEventListener('error', () => console.log('Transaction error'));
}
```

## Mostrar registros

Para mostrarlos, recorre todos los valores en el cursor y los agrega con elementos HTML a todos (el div que contiene la `ol`). Se le agrega un atributo `data-id` para luego poder borrarlos.

```javascript
function showTodos() {
  while (todos.firstChild) {
    todos.removeChild(todos.firstChild);
  }
  const objectStore = db.transaction('todo_tb').objectStore('todo_tb');
  objectStore.openCursor().addEventListener('success', e => {

    const pointer = e.target.result;
    if(pointer) {
      const listItem = document.createElement('li');
      const h3 = document.createElement('h3');
      const pg = document.createElement('p');
      listItem.appendChild(h3);
      listItem.appendChild(pg);
      todos.appendChild(listItem);
      h3.textContent = pointer.value.title;
      pg.textContent = pointer.value.body;
      listItem.setAttribute('data-id', pointer.value.id);
      const deleteBtn = document.createElement('button');
      listItem.appendChild(deleteBtn);
      deleteBtn.textContent = 'Remove';
      deleteBtn.addEventListener('click', deleteItem);
      pointer.continue();
    } else {
      if(!todos.firstChild) {
        const listItem = document.createElement('li');
        listItem.textContent = 'No Todo.'
        todos.appendChild(listItem);
      }

      console.log('Todos all shown');
    }
  });
}
```

## Eliminar registros

Busca el elemento con `data-id` para borrar del object-store y después eliminar dicho elemento del arbol DOM.

```javascript
function deleteItem(e) {
  const todoId = Number(e.target.parentNode.getAttribute('data-id'));
  const transaction = db.transaction(['todo_tb'], 'readwrite');
  const objectStore = transaction.objectStore('todo_tb');
  objectStore.delete(todoId);
  transaction.addEventListener('complete', () => {
    e.target.parentNode.parentNode.removeChild(e.target.parentNode);
    alert(`Todo with id of ${todoId} deleted`)
    console.log(`Todo:${todoId} deleted.`);
    if(!todos.firstChild) {
      const listItem = document.createElement('li');
      listItem.textContent = 'No Todo.';
      todos.appendChild(listItem);
    }
  });
  transaction.addEventListener('error', () => console.log('Transaction error'));
}
```

## Versiones de IndexedDB

Al crear la DB, se especifica la versión de IndexedDB (En este caso, 1). Si la bbdd no existe, se crea. Si ya existía, se comprueba el número de versión. Si el número de versión es mayor que el que ya existe, se lanza un evento onUpgradeNeeded que permite hacer la migración o los cambios necesarios en la bbdd. Al migrar, hay que tener cuidado, porque borrar una tabla para volver a crearla (por ejemplo, porque tiene nuevas columnas) hace que se borren los datos, así que habría que almacenarlos previamente en otro sitio.
