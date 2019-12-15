<!-- .slide: class="titulo" -->
# Arquitecturas de *apps* iOS. Parte II: **Algunas alternativas a MVC**

---

## Puntos a tratar

- **MVVM (Model/View/ViewModel)**
- VIPER
- Redux

---

## MVVM (Model/View/ViewModel)

![](img/mvvm.png) <!-- .element class="stretch"-->

La parte "nueva" es el *viewmodel*, que se encarga de la *lógica de presentación*, es decir convertir/modificar/formatear datos (p. ej. fechas, distancias, ...) en el formato adecuado para la vista

A cambio se elimina el *controller*, sustituido en parte por el viewmodel y en parte por los *bindings* entre este y la vista

---

## Más sobre el viewmodel

![](img/mvvm.png) <!-- .element class="stretch"-->

* El *viewmodel* es independiente de la tecnología de la vista (no hay un `import UIKit`)
* **bindings entre viewmodel y vista** Cuando cambia un "lado", el otro lo hace también automáticamente

---

## Más sobre la vista

![](img/mvvm.png) <!-- .element class="stretch"-->

- Lo mismo que era en MVC, pero...
- Aunque pueda parecer un poco raro, un "view controller" de iOS también se considera vista

---

## Por qué un ViewController debería ser vista

- Al estar el ciclo de vida del "view controller" y de los elementos de interfaz tan unidos, es mejor considerarlos a todos como vista
- Además conseguimos que la vista sea el único componente directamente dependiente de la tecnología de presentación, en iOS `UIKit`

---

## *Bindings*

- Necesitamos alguna librería/*framework* que implemente estos *bindings*, ya que en iOS no existen de forma nativa (aunque sí en OSX)
- Aquí usaremos una librería llamada [Bond](https://github.com/ReactiveKit/Bond), (Swift Bond, :) ). Hay otras alternativas: ReactiveCocoa, RxSwift, ...

---

## *Data binding* en Bond

- Se pueden vincular componentes de usuario en la vista (`Label`, `TextField`,...) con *observables* en el viewmodel. 
- **Observable** es un concepto tomado de la *programación funcional reactiva* (también llamados *streams*, *signals*,...)
- Similares a los eventos. Uno o más observadores (similares a *listeners*), reciben los cambios en el valor del observable
- La parte de programación funcional es la que nos permite transformar/manipular/combinar los observables

---

## Observables en Bond

- En el *ViewModel* podemos crear un observable del tipo necesario

```swift
var obs = Observable<String>("")
```

- `obs` no es un `String`, sino un observable que "produce" `String`s. El `String` está en `obs.value`

```swift
obs.value = "un nuevo valor"
```

---

## *Binding* del observable

- Lo más típico es vincular el observable con una propiedad de un control de `UIKit`. Bond extiende los controles con una propiedad `reactive`, y dentro de ella los campos habituales del control (`text`, `textColor`,...) pero en versión *observable*

```swift
//suponemos un outlet en la vista que representa un "label": labelOutlet
viewModel.obs.bind(to:self.labelOutlet.reactive.text)
```

---

## UAdivino versión MVVM 

- Repo Github: [https://github.com/ottocol/ejemplos-arquitectura-iOS/tree/master/MVVM/UAdivino](https://github.com/ottocol/ejemplos-arquitectura-iOS/tree/master/MVVM/UAdivino)

---

## Ensamblando Vista/ViewModel/Modelo

- Recordar que el *view controller* es parte de la vista. En él definimos
```swift
class UAdivinoView : UIViewController { 
   let viewModel = UAdivinoViewModel()
   ...
}
```

- En el view model
```swift
class UAdivinoViewModel {
   let model = UAdivinoModel()
   ...
}
```

---

## Mostrar el texto de la respuesta en la vista

*binding* entre el observable `textoResp` del viewmodel y un *label* de la vista

- En el view model

```swift
let textoResp = Observable<String>("")
...
let resp = model.obtenerRespuesta()
textoResp.value = resp.texto
```

- En la vista

```swift
override func viewDidLoad() {
    self.viewModel.textoResp.bind(to:self.respuestaLabel.reactive.text)
}
```

---

## Binding de la vista al viewmodel

```swift
//En la vista
//Lo que escribamos en el campo se coloca en la propiedad (debe ser un observable)
self.campoNombre.reactive.text.bind(to: viewModel.nombreUsuario)
```

```swift
//En el viewmodel
let nombreUsuario = Observable<String?>("")
```


---

## Cambiar el color de la respuesta en la vista


- Problema: En el *view model* hemos definido el color como un enumerado, pero en la vista es un `UIColor`, que es como se representan los colores en `UIKit
. No podemos vincularlos porque son de tipos distintos
- Tampoco deberíamos usar `UIColor` en el *view model*, para preservar la independencia de `UIKit`. Usamos un enumerado propio


**¿Y ahora qué?**

---

## Transformar los observables

- **Programación Funcional Reactiva**: aplicando primitivas típicas de programación funcional, podemos transformar los valores que "emite" un observable

```swift
//En la vista
self.viewModel.colorResp
    //con esto filtramos todos los valores que no sean verde o rojo
    .filter {
        color in
        return (color == .verde || color == .rojo) ? true : false
    }
    //con esto transformamos valores de enumerado a UIColor
    .map {
       color in
       return (color == .verde ? UIColor.green : UIColor.red)
    }
    //Ahora ya podemos hacer el binding
    .bind(to: self.labelRespuesta.reactive.textColor)
```


---

## Puntos a tratar

- MVVM (Model/View/ViewModel)
- **VIPER**
- Redux

---

## VIPER

- "Un paso más", ya que ninguna arquitectura MVx
    + Detalla cómo estructurar el modelo
    + Habla sobre navegación (cambio de pantallas)
- Adaptación de la Clean Architecture de "Uncle" Bob Martin

![](https://techbeacon.com/sites/default/files/styles/article_hero_image/public/robert-uncle-bob-martin-agile-manifesto-interview.jpg)

---

![](https://8thlight.com/blog/assets/posts/2012-08-13-the-clean-architecture/CleanArchitecture-8b00a9d7e2543fa9ca76b81b05066629.jpg)

- [https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)

---

## VIPER

- View, Interactor, Presenter, Entity, Router 
- [https://www.objc.io/issues/13-architecture/viper/](https://www.objc.io/issues/13-architecture/viper/)

![](img/viper.png)

---

## Principios básicos que subyacen a VIPER

- Single Responsibility
- "Program to interfaces, not implementations"
- Dependency inversion

---

## Problema fundamental de VIPER

- Demasiada "infraestructura": por cada caso de uso o módulo
    + 5 componentes (V, I, P, E, R)
    + 2 interfaces por componente

- Generadores de plantillas VIPER
    + https://github.com/pepibumur/viper-module-generator
    + https://github.com/rambler-digital-solutions/Generamba
    + https://github.com/ferranabello/Viperit

---

## VIPER "en acción"

- La app de ejemplo original (ahora en Swift) [https://github.com/mutualmobile/VIPER-SWIFT](https://github.com/mutualmobile/VIPER-SWIFT)

![](img/viper_todo.png)

---

## Puntos a tratar

- MVVM (Model/View/ViewModel)
- VIPER
- **Redux**

---

## Redux

- Nacido en el mundo web (React) e importado a otros contextos
- `SwiftUI`, la alternativa de Apple a `UIKit` toma muchas ideas del estilo React/Redux
- Lo veremos con más detalle

![](img/reswift_detail.png) <!-- .element: class="stretch" -->




---

## Más referencias sobre arquitecturas en iOS

La "lista maestra": [https://github.com/onmyway133/fantastic-ios-architecture](https://github.com/onmyway133/fantastic-ios-architecture)

---

# ¿Alguna pregunta?