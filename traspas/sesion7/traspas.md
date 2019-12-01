<!-- .slide: class="titulo" --> 

# Sesión 7: Tablas en Core Data
## Persistencia en dispositivos móviles, iOS


---

## Puntos a tratar

- ***Fetched results controller***
- Mostrar los datos en la tabla
- Actualizar automáticamente la tabla
- Secciones de tabla


---

## ¿De dónde vienen los datos en una tabla?

- Típicamente de un `Array`. Con Core Data los sacamos con una *fetch request* para meterlos en el array
- En el objeto que actúa como *datasource*:

```swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    //recordar que el prototipo de celda tiene un "reuse identifier"
    //que hay que asignar en el storyboard
    let cell = tableView.dequeueReusableCell(withIdentifier: "miCelda", for: indexPath)
    let mensaje = mensajes[indexPath.row]
    cell.textLabel?.text = mensaje.texto!
    return cell
}
```

---

## Problemas

- Si hay muchos datos es **ineficiente** tenerlos todos en memoria, sería mejor ir sacándolos según nos movamos por la tabla, pero no es tan trivial de implementar
- si se actualiza algún dato en Core data o bien
    + Hacemos otra vez la *request* y recargamos toda la tabla
    + Manualmente lo añadimos al array


---

## `NSFetchedResultsController`

- Sirve de **"puente"** entre Core Data y la vista de tabla
- Mejora el rendimiento:
  - Obtiene solo los datos necesarios, los que se están mostrando
  - Puede guardar automáticamente los datos en una *cache* 
- Puede **detectar los cambios en el contexto** para que podamos actualizar la tabla
- Nos ayuda a crear **secciones** automáticamente 

---

## Un `NSFetchedResultsController` básico

- Para inicializarlo necesitamos
  + Asociarle una *fetch request* que obtenga los datos a mostrar. 
  + La *request* debe estar **ordenada** (Usar `NSSortDescriptor`s)
  
```swift  
let miDelegate = UIApplication.shared.delegate! as! AppDelegate
let miContexto = miDelegate.persistentContainer.viewContext

let consulta = NSFetchRequest<Mensaje>(entityName: "Mensaje")

let sortDescriptors = [NSSortDescriptor(key:"fecha", ascending:false)]
consulta.sortDescriptors = sortDescriptors
let frc = NSFetchedResultsController<Mensaje>(fetchRequest: consulta, managedObjectContext: miContexto, sectionNameKeyPath: nil, cacheName: "miCache")

//ejecutamos el fetch
try! frc.performFetch()
```


---

Podemos guardar el *fetched results controller* en el `ViewController` de la pantalla con la tabla, sup. un `UITableViewController`.

Cambiaríamos la variable `frc` de la traspa anterior por `self.frc`

```swift
import UIKit
import CoreData

class MiController : UITableViewController {
  var frc : NSFetchedResultsController<Mensaje>! 
  ...
}
```

Así, este *controller* hace de todo (cosa que no debería :))
- almacena el `frc`
- es el *delegate* de la tabla
- es el *datasource* de la tabla


---

## Puntos a tratar

- *Fetched results controller*
- **Mostrar los datos en la tabla**
- Actualizar automáticamente la tabla
- Secciones de tabla


---

## Para ver los datos en la tabla

- Recordemos que debe haber un objeto que actúe de *datasource* 
- El *datasource* debe implementar
  -  `numberOfSections(in:)` devuelve número de secciones
  -  `tableView:numberOfRowsInSection:` devuelve número de filas en una sección
  -  `tableView:cellForRowAtIndexPath:` devuelve una celda

---

## Número de secciones

Simplemente lo obtenemos del *fetched results controller*

```swift
override func numberOfSections(in tableView: UITableView) -> Int {
    return self.frc.sections!.count
}
```

---

## Número de filas en la sección actual:

Idem, lo tomamos del *fetched results controller*

```swift
override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return self.frc.sections![section].numberOfObjects
}
```

---

## Obtener una celda, dada la fila


```swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    //recordar que el prototipo de celda tiene un "reuse identifier"
    //que hay que asignar en el storyboard
    let cell = tableView.dequeueReusableCell(withIdentifier: "miCelda", for: indexPath)
    
    let mensaje = self.frc.object(at: indexPath)
    cell.textLabel?.text = mensaje.texto!
    return cell
}
```

---

## Puntos a tratar

- *Fetched results controller*
- Mostrar los datos en la tabla
- **Actualizar automáticamente la tabla**
- Secciones de tabla

---

## Refrescar la tabla

- De momento igual que con el array: cada vez que creamos un nuevo objeto este no aparece en la tabla por sí solo, hay que llamar a `reloadData()` 
- PERO: el *fetched results controller* está “suscrito” a los cambios que se producen en el contexto de persistencia
- Para que nos avise a su vez de estos cambios, tenemos que convertirnos en su *delegate*. 

```swift
self.frc.delegate = self;
```

---

## Convertirse en el *delegate*

1. Asignar valor a la propiedad `delegate` del *fetched results controller* (ya hecho)
2. Declarar que la clase es conforme al protocolo `NSFetchedResultsControllerDelegate`

```swift
import UIKit
import CoreData

class MiController : UITableViewController, NSFetchedResultsControllerDelegate {
 ...
}
```

---

## "Escuchar" los cambios

Cuando se van a modificar los datos y cuando ya se han modificado el *fetched results controller* avisará a su *delegate* llamando a `controllerWillChangeContent` y `controllerDidChangeContent`. Aprovechamos para llamar a `beginUpdates()` y `endUpdates` de la tabla

```swift
func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    self.tableView.beginUpdates()
}

func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    self.tableView.endUpdates()
}
```

Con esto, agrupamos todos los cambios en una sola animación

---

## Visualizar los cambios

Cuando se ha modificado algún objeto del contexto bajo la supervisión del *fetched results controller* avisará a su delegate llamando a:

```swift
func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange anObject: Any, at indexPath: IndexPath?, for type: NSFetchedResultsChangeType, newIndexPath: IndexPath?) {
    switch type {
    case .insert:
        self.tableView.insertRows(at: [newIndexPath!], with:.automatic )
    case .update:
        self.tableView.reloadRows(at: [indexPath!], with: .automatic)
    case .delete:
        self.tableView.deleteRows(at: [indexPath!], with: .automatic)
    case .move:
        self.tableView.deleteRows(at: [indexPath!], with: .automatic)
        self.tableView.insertRows(at: [newIndexPath!], with:.automatic )
    }
}
```

---

## Modificación en las secciones

El *fetched results controller* también avisa a su *delegate* cuando se modifican las secciones

```swift
func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>, didChange sectionInfo: NSFetchedResultsSectionInfo, atSectionIndex sectionIndex: Int, for type: NSFetchedResultsChangeType) {
    switch(type) {
    case .insert:
        self.tableView.insertSections(IndexSet(integer:sectionIndex), with: .automatic)
    case .delete:
        self.tableView.deleteSections(IndexSet(integer:sectionIndex), with: .automatic)
    default: break
    }
}
```

---

## Puntos a tratar

- *Fetched results controller*
- Mostrar los datos en la tabla
- Actualizar automáticamente la tabla
- **Secciones de tabla**

---

## Crear secciones automáticamente

Con el atributo `sectionNameKeyPath` al inicializar el *fetched results controller*

```swift
self.frc = NSFetchedResultsController<Mensaje>(fetchRequest: consulta, managedObjectContext: miContexto, sectionNameKeyPath: "conversacion.titulo", cacheName: "miCache")
```

---

## "Pintar" los títulos de sección

Recordemos que el sitio de donde se sacan los datos es el *datasource*, al que la tabla le pedirá no solo las celdas sino los títulos de sección

tabla -> Datasource -> fetched results controller

```swift
override func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
    return self.frc.sections![section].name
}
```

---


# ¿Alguna pregunta?