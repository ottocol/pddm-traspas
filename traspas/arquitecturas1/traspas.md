
<!-- .slide: class="titulo" -->
# Arquitecturas de aplicaciones iOS. **Parte I: Introducción. MVC**



---

## Puntos a tratar

- **Arquitectura de aplicaciones iOS**
- Problemas de MVC/posibles soluciones
- Flujo de información en la app


---

Hasta ahora nos hemos preocupado por APIs y tecnologías, pero no demasiado por cómo **estructurar el código de la aplicación**

---

## ¿Qué debe tener una buena arquitectura?

Entre otras cosas...

- Cada clase debe desempeñar un único papel 
- Debe facilitar el testing 
- No debe depender de un framework concreto 
- Debe ser flexible por simplicidad pero no por demasiada abstracción 
- Debe permitir seguir el flujo de datos con facilidad 

[Charla "Good iOS Application Architecture: MVVM vs. MVC vs. VIPER"](https://academy.realm.io/posts/krzysztof-zablocki-mDevCamp-ios-architecture-mvvm-mvc-viper/
) <!-- .element: class="caption" -->


---

<!-- .slide: data-background-image="https://www.mycustomer.com/sites/default/files/styles/banner/public/warning_0_0_0_0_1_0_0.jpg" -->

<span style="color:red; font-weight:bolder; font-size: 2em">A partir de aquí, todo es opinable</span>


---

## Puntos a tratar

- Arquitectura de aplicaciones iOS
- **Problemas de MVC/posibles soluciones**
- Flujo de información en la app



---

## MVC en la teoría

![](img/expectativa_mvc.png)

---

## Problema: Acoplamiento entre componentes

Por el propio diseño de iOS hay un acoplamiento vista/controlador

![](img/realidad_mvc.png)

Este problema lo resuelven otras arquitecturas como MVVM

---

## Evitar acoplamiento entre modelo y controlador

El controlador puede guardar referencias al modelo

```swift
class MiViewController : ViewController {
    var miModelo : Modelo
    ...
}
```

pero no al revés, ya que **el modelo no sería reutilizable**

```swift
class Modelo  {
    //NOOOOOOOOOOO!!!!!
    var mc : MiViewController
    ...
}
```

---

Recordad que en iOS hay dos mecanismos básicos que podemos usar para independizar el modelo del controlador:

- Notificaciones
- KVO

El controlador siempre debe **escuchar al modelo**, no el modelo enviarle específicamente los datos al controlador

---

## Problema: Massive View Controllers


<div class="column half">
    ![](img/tweet_mvc.png)
</div>
<div class="column half">    
![](img/family_mvc.jpg)
</div>


---

Ejemplo de aplicación en la que "todo lo hace el view controller"

- [Fuente de la *app* en Github](https://github.com/ottocol/mvc-refactor-swift/) (versión actual ya refactorizado)
- [Código del *view controller*](https://github.com/ottocol/mvc-refactor-swift/blob/v1.0/ListaCompra/ListaViewController.swift)

---

## ¿Qué funciones está haciendo aquí el controller?


---

## Vamos a refactorizar el *view controller* para que no realice tantas tareas distintas


---

## Fuentes

- Principios básicos de programación
- Patrones de diseño
- "Sentido común"

---


### **Principio de Responsabilidad Única**: una clase debería tener solo una razón para cambiar

Nuestro *view controller* debe cambiar si cambia:

- La agrupación de las tareas en categorías
- El almacenamiento (p.ej. guardarlo como JSON, o como preferencias)
- La representación de los datos en la tabla
- ...

Otro principio similar: separación de intereses (*separation of concerns*)

---

## Cambios a realizar

* Encapsular la lógica en un **modelo** 
*  Encapsular la persistencia en un **repositorio** (también llamado DAO, *data mapper*, ...)
    * Además de separar responsabilidades, facilita el cambio en el mecanismo de persistencia (¿preferencias?, ¿SQLite?)
* Separar la responsabilidad de actuar como **datasource**
* Separar la responsabilidad de actuar como **delegate**                
* Extraer el código que **configura las celdas** (encapsular la vista)

---

## Encapsular la persistencia

Ya vimos un ejemplo en la sesión de SQLite

```swift
class DBManager {
    var db : OpaquePointer? = nil
    
    init(conDB nombreDB : String) {
        ...
    }

    func listarTareas() -> [Tarea] {
        ...
    }
```

---

## Encapsular la persistencia en una *app* con Core Data

Es más complicado, ya que Core Data mezcla **modelo** (entidades) y **persistencia**. Las entidades están íntimamente ligadas al contexto de persistencia

```swift
let u = Usuario(context:miContexto)
```

No es un problema tan grande, porque...

<ul>
<li class="fragment">El API de persistencia es suficientemente sencillo como para "no molestar"</li>
<li class="fragment">En una *app* con Core Data raramente cambiaremos de API de persistencia</li>
</ul>

---

## No obstante, si nos empeñáramos...

![](img/dtos.png)

Los **Data Transfer Objects** (DTOs) son copias de las entidades, pero sin estar vinculados a ningún contexto de persistencia ([patrón de diseño](https://martinfowler.com/eaaCatalog/dataTransferObject.html) típico de *apps enterprise*)

Así, tendríamos una clase `UsuarioDTO` y una clase `Usuario` que sería la entidad de Core Data. Nuestro código trabajaría solo con la primera, la segunda la vería solo el repositorio

---

## Separar el Datasource del ViewController

- [ListaCompraDataSource](https://github.com/ottocol/mvc-refactor-swift/blob/master/ListaCompra/ListaCompraDataSource.swift)
- [ListaViewController](https://github.com/ottocol/mvc-refactor-swift/blob/master/ListaCompra/ListaViewController.swift)

---

## Separar la celda del DataSource

Siendo puristas, una celda es parte de la *vista*, por lo que aquí estamos mezclando responsabilidades

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let celda = tableView.dequeueReusableCell(withIdentifier: "MiCelda", for: indexPath)
        if let prioridad = Prioridad(rawValue: indexPath.section),
            let item = lista.getItem(pos: indexPath.row, prioridad: prioridad) {
            celda.textLabel?.text = item.nombre
            if item.comprado {
                celda.accessoryType = .checkmark
            }
            else {
                celda.accessoryType = .none
            }
        }
        return celda
}
```

---

## Encapsular la celda en su propia clase

```swift
class CeldaItem : UITableViewCell {
    static let nombre = "MiCelda"
    
    var nombre : String? {
        didSet {
            self.textLabel?.text = self.nombre
        }
    }
    
    var comprado : Bool? {
        didSet {
            self.accessoryType = self.comprado! ? .checkmark : .none
        }
    }
}
```
[CeldaItem.swift](https://github.com/ottocol/mvc-refactor-swift/blob/master/ListaCompra/vista/CeldaItem.swift) <!-- .element: class="caption" -->

---


Para más detalles, podéis ver la charla: "*Refactoring tne Mega Controller*", de Andy Matuschak (**¡Muy recomendable!**)

- [Video de la charla](https://vimeo.com/140037432)
- [Código de ejemplo en Github](https://github.com/andymatuschak/refactor-the-mega-controller)


---

## Puntos a tratar

- Arquitectura de aplicaciones iOS
- Problemas de MVC/posibles soluciones
- **Flujo de información en la app**



---

## Problema: MVC no es una arquitectura para toda la *app*

MVC cubre "una pantalla", pero ¿qué pasa al cambiar de una pantalla a otra?
- Ya hemos visto que necesitamos pasar datos de un `ViewController` a otro
- Además necesitamos saber a qué pantallas se puede ir desde una dada, y cómo 

---

## Pasar datos entre controllers

+ *Singleton*
+ Inyección de dependencias

---

## Singleton

La versión "civilizada" de la variable global. Omnipresente en iOS (*app delegate*, *notification center*, cola principal, preferencias de usuario,...) 

```swift
//Definición
class StateSingleton {
    var pedidoActual:Pedido!
    
    private init(){
    }
    
    static let shared = StateSingleton()
}
```
```swift
//Uso
StateSingleton.shared.pedidoActual
```

---

## Inyección de dependencias

La hemos usado varias veces

```swift
//En el view controller 1
override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
    if segue.identifier=="siguiente" {
        if let vc2 = segue.destination as? ViewController2 {
            vc2.mensaje = "Bienvenidos a la pantalla 2"
        }
    }
 }
```
```swift
//En el view controller 2
class ViewController2 : UIViewController {
    var mensaje : String!
    ...
}
```
Problema: no tenemos ninguna ayuda del compilador para asegurarnos de que estamos pasando los datos 

---

Desde iOS13 *segue action*, un método que iOS llama durante la transición de un *segue* para que podamos crear nosotros el view controller, en vez de crearlo iOS. Nos permite usar un *inicializador a medida*, **cuyo uso correcto chequea el compilador**

```swift
@IBSegueAction
private func showPreview(coder: NSCoder)
    -> ViewController2? {
    return ViewController2(coder: coder, mensaje: "hola")
}
```

```swift
import UIKit
class ViewController2: UIViewController {
  let mensaje: Mensaje

  init?(coder: NSCoder, mensaje: texto) {
    self.mensaje = texto
    super.init(coder: coder)
  }
  ...
}
```

Más detalles por ejemplo en [Better Storyboards  with Xcode 11](https://useyourloaf.com/blog/better-storyboards-with-xcode-11/)

---

## Flujo de navegación

¿Cómo especificar el flujo de navegación entre pantallas de forma que sea fácil de entender, escalable y compatible con la gestión de versiones?

- *Storyboards* 
    - Problemas de escalabilidad. Mejor dividir el storyboard en fragmentos con *storyboard references* ([tutorial ejemplo](https://cocoacasts.com/organizing-storyboards-with-storyboard-references))
    - Al ser XML generado automáticamente plantean [problemas con el control de versiones](https://martiancraft.com/blog/2018/02/handling-storyboard-merge-conflicts/)
- Arquitecturas como VIPER intentan solucionar (entre otros) este problema



---

# ¿Alguna pregunta?