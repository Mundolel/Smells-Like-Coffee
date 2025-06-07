Cuando tú escribes

```java
new Image("/bro/brocode/test.png");
```

JavaFX espera que la cadena que le pasas sea **directamente** una URL (por ejemplo `file:/C:/iconos/test.png` o `http://…`). Como no existe ningún fichero en tu disco duro con ruta absoluta `/bro/brocode/test.png`, simplemente no lo carga.

En cambio, cuando usas:

```java
URL iconUrl = HelloApplication.class.getResource("test.png");
Image icon = new Image(iconUrl.toExternalForm());
```

estás haciendo dos cosas diferentes:

1. **Localizar el recurso en el *classpath***

   * `HelloApplication.class.getResource("test.png")`
     Busca `test.png` **relativo al paquete** de la clase. Como tu clase está en `bro/brocode/`, en tiempo de ejecución el *classpath* incluye algo como `…/target/classes/`, y dentro de ahí existe `bro/brocode/test.png`.

     * Si pones una barra al principio, `getResource("/bro/brocode/test.png")`, entonces busca “desde la raíz del classpath” ese fichero.
   * El resultado es un objeto `URL` que apunta internamente al recurso empaquetado dentro del JAR (o dentro de `target/classes` cuando estás en desarrollo).

2. **Convertir esa ubicación en algo que JavaFX entienda**

   * `iconUrl.toExternalForm()` produce una cadena como

     ```
     "file:/C:/…/target/classes/bro/brocode/test.png"
     ```

     o, cuando empaquetas en un JAR, algo como

     ```
     "jar:file:/…/mi-app.jar!/bro/brocode/test.png"
     ```
   * Al pasársela a `new Image(...)`, JavaFX puede abrir esa URL y leer los bytes de la imagen.

---

## Rutas relativas vs. rutas absolutas en `getResource`

* **Relativa (sin `/` al inicio)**

  ```java
  getClass().getResource("archivo.ext");
  ```

  → Busca dentro del mismo paquete que la clase.
  Ejemplo: tu clase `bro.brocode.HelloApplication` busca en `bro/brocode/`.

* **Absoluta (con `/` al inicio)**

  ```java
  getClass().getResource("/ruta/completa/archivo.ext");
  ```

  → Busca desde la raíz del classpath.
  Ejemplo: `"/bro/brocode/test.png"` busca exactamente en `…/target/classes/bro/brocode/test.png`.

---

## Por qué tu cadena literal falla

Cuando escribes

```java
new Image("/bro/brocode/test.png");
```

no hay ningún mecanismo de “búsqueda en el classpath”. JavaFX lo toma como si tú mismo hubieras escrito:

```java
new Image("file:/bro/brocode/test.png");
```

y, claro, no existe. No hay `test.png` en la raíz de tu disco en la ruta `/bro/brocode/`.

---

## Cómo pensar en ello

1. **Classpath** = “el cajón donde Java busca clases y recursos empaquetados.”
2. **getResource / getResourceAsStream** = “Oye, búscame este recurso **dentro de ese cajón**, ya sea en la carpeta de mi paquete o desde la raíz.”
3. **new Image(URL)** = “Ok, esto ya es una dirección que entiendo: ábrela y lee la imagen.”
4. **new Image(String)** = “Trátalo como una URL externa, no como un recurso empaquetado.”

---

### Resumen práctico

* **Para recursos empaquetados** (FXML, iconos, archivos de texto…)

  ```java
  URL url = MyClass.class.getResource("algo.ext");       // ruta relativa
  // —o—
  URL url = MyClass.class.getResource("/paquete/algo.ext"); // ruta absoluta
  Image img = new Image(url.toExternalForm());
  ```
* **Para rutas del sistema de archivos** (p. ej. `C:\iconos\otro.png`)

  ```java
  File file = new File("C:/iconos/otro.png");
  Image img = new Image(file.toURI().toString());
  ```
* **Para streams** (útil si quieres leer vía InputStream)

  ```java
  InputStream in = MyClass.class.getResourceAsStream("algo.ext");
  Image img = new Image(in);
  ```

Con esto tienes cubierto cómo y **por qué** cada forma funciona (o no) según de dónde venga tu imagen.


Cuando tú pasas una cadena a

```java
new Image(String url);
```

JavaFX no va a buscar en tu *classpath*, sino que interpreta esa cadena **directamente** como una URL. Por eso si pones algo como `"/bro/brocode/test.png"` lo toma como el path absoluto del sistema (`file:/bro/brocode/test.png`), y claro ese fichero no existe en tu disco.

---

### ¿Qué ruta tendrías que darle?

De las que muestra IntelliJ:

| Ruta desde…        | Ejemplo                                           |
| ------------------ | ------------------------------------------------- |
| **Contenido raíz** | `src/main/resources/bro/brocode/test.png`         |
| **Source root**    | `bro/brocode/test.png`                            |
| **Absolute Path**  | `C:\Users\PC\IdeaProjects\broCode\src\…\test.png` |

**Si vas a usar `new Image(String)` SIN `getResource()`, necesitas la ruta absoluta convertida a URI de archivo**. Por ejemplo:

```java
String absolute = "C:/Users/PC/IdeaProjects/broCode/src/main/resources/bro/brocode/test.png";
Image icon = new Image("file:///" + absolute.replace("\\", "/"));
```

o bien (más estándar):

```java
File f = new File("C:\\Users\\PC\\IdeaProjects\\broCode\\src\\main\\resources\\bro\\brocode\\test.png");
Image icon = new Image(f.toURI().toString());
```

Eso genera algo como

```
file:///C:/Users/PC/IdeaProjects/broCode/src/main/resources/bro/brocode/test.png
```

y JavaFX lo abrirá como un fichero del sistema.

---

### ¿Por qué no es recomendable?

1. **Rompe el empaquetado:** si luego empacas tu app en un JAR, esa ruta absoluta deja de tener sentido.
2. **No es portable:** otro equipo tendrá distinto `C:\…`.
3. **Pierdes la flexibilidad** de los recursos internos de tu JAR.

Por eso lo habitual es seguir usando

```java
URL url = MyClass.class.getResource("test.png");      // dentro del JAR
Image icon = new Image(url.toExternalForm());
```

o

```java
Image icon = new Image(MyClass.class.getResourceAsStream("test.png"));
```

y así tu icono siempre viaja dentro del JAR, sin depender de rutas fijas en disco.
