# Prueba de traspas con RevealJS
## iOS, sesión X

---

## Vistas y jerarquía de vistas

- cada vista está asociada a un controller, como ya sabemos, y tiene subvistas

```swift
//este código estaría en un controller
let boton = UIButton(type: .system)
boton.setTitle("Soy un botón", for: .normal)
boton.frame = CGRect(x: 100, y: 100, width: 100, height: 50)
self.view.addSubview(boton)
```

