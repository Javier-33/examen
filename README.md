# Examen
## Javier Viñan

Se hace los imports necesarios:
```scala
// Para trabajar con bases de datos en Scala

import doobie._
import doobie.implicits._

// Manejo de efectos (IO) de forma segura
import cats.effect._
import cats.implicits._

// Para leer archivos CSV desde el sistema
import scala.io.Source
```

### Cada atributo corresponde a una columna de la base de datos.
Representa una fila del csv.

```scala
case class Ataque(
                   id: Int,                            
                   tipoAtaque: String,                 
                   ipOrigen: String,                
                   ipDestino: String,                    
                   severidad: String,                     
                   paisOrigen: String,                   
                   paisDestino: String,                   
                   fechaAtaque: String,                   
                   duracionMinutos: Int,                  
                   contenidoSensibleComprometido: Boolean 
                 )
```

Aqui se conecta a el MySql mediante el usuario y contraseña.

```scala
object Examen extends IOApp.Simple {
  val xa = Transactor.fromDriverManager[IO](
    driver = "com.mysql.cj.jdbc.Driver",
    url = "jdbc:mysql://localhost:3306/examen",
    user = "root",
    password = "Javisofy.33",
    logHandler = None
  )
```


Desde el codigo de Scala crea la tabla MySql ataques si no existe.

```scala
  def crearTabla(): ConnectionIO[Int] =
    sql"""
      CREATE TABLE IF NOT EXISTS ataques (
        id INT PRIMARY KEY,
        tipo_ataque VARCHAR(80),
        ip_origen VARCHAR(45),
        ip_destino VARCHAR(45),
        severidad VARCHAR(10),
        pais_origen VARCHAR(80),
        pais_destino VARCHAR(80),
        fecha_ataque VARCHAR(30),
        duracion_minutos INT,
        contenido_sensible_comprometido BOOLEAN
      )
    """.update.run
```

Borra todos los registros existentes para evitar duplicados al volver a cargar el CSV.

```scala
  def limpiarTabla(): ConnectionIO[Int] =
    sql"TRUNCATE TABLE ataques".update.run
```

Inserta una fila en la tabla ataques.

```scala
  def insertAtaque(
                    id: Int,
                    tipoAtaque: String,
                    ipOrigen: String,
                    ipDestino: String,
                    severidad: String,
                    paisOrigen: String,
                    paisDestino: String,
                    fechaAtaque: String,
                    duracionMinutos: Int,
                    contenidoSensible: Boolean
                  ): ConnectionIO[Int] =
    sql"""
      INSERT INTO ataques
      (id, tipo_ataque, ip_origen, ip_destino, severidad,
       pais_origen, pais_destino, fecha_ataque,
       duracion_minutos, contenido_sensible_comprometido)
      VALUES
      ($id, $tipoAtaque, $ipOrigen, $ipDestino, $severidad,
       $paisOrigen, $paisDestino, $fechaAtaque,
       $duracionMinutos, $contenidoSensible)
    """.update.run
```

Recupera todos los registros de la tabla ataques y los mapea automáticamente al case class Ataque.

```scala
  def listAllAtaques(): ConnectionIO[List[Ataque]] =
    sql"""
      SELECT
        id,
        tipo_ataque,
        ip_origen,
        ip_destino,
        severidad,
        pais_origen,
        pais_destino,
        fecha_ataque,
        duracion_minutos,
        contenido_sensible_comprometido
      FROM ataques
    """.query[Ataque].to[List]
```

Metodo Run

```scala
  override def run: IO[Unit] = {
```


Se lee el archivo CSV desde la ruta indicada.

```scala

    val lineasCsv =
      Source
        .fromFile("/Users/javiervinan/IdeaProjects/ProyectoFinal/src/main/resources/data/ataques.csv") //Mi ruta desde el Path
        .getLines()
        .drop(1)
        .toList
```

Cada línea del CSV se divide por comas y se envía como parámetros al método insertAtaque.

```scala
    val operacionesDeInsercion = lineasCsv.map { linea =>
      val cols = linea.split(",")

      insertAtaque(
        cols(0).trim.toInt,                    // ID
        cols(1).trim,                          // Tipo_Ataque
        cols(2).trim,                          // IP_Origen
        cols(3).trim,                          // IP_Destino
        cols(4).trim,                          // Severidad
        cols(5).trim,                          // País_Origen
        cols(6).trim,                          // País_Destino
        cols(7).trim,                          // Fecha_Ataque
        cols(8).trim.toInt,                    // Duración en minutos
        cols(9).trim.equalsIgnoreCase("true")  // Contenido sensible
      )
    }
```


Aquí se arma el flujo de ejecución:
    
```scala
    val programa = for {
      _     <- crearTabla()
      _     <- limpiarTabla()
      _     <- operacionesDeInsercion.sequence
      todos <- listAllAtaques()
    } yield todos
```

transact(xa) ejecuta el programa usando la conexión definida.

```scala
    programa.transact(xa).flatMap { todos =>
      IO.println("\n--- ATAQUES REGISTRADOS ---") *>
        IO(
          todos.foreach(a =>
            println(
              f"${a.id}%2d | ${a.tipoAtaque}%-25s | ${a.ipOrigen} -> ${a.ipDestino} | ${a.severidad} | ${a.paisOrigen} -> ${a.paisDestino} | ${a.fechaAtaque} | ${a.duracionMinutos}%3d min | ${a.contenidoSensibleComprometido}"
            )
          )
        ) *>
        IO.println(s"\nTotal de ataques: ${todos.size}")
    }
  }
}
```

## RESULTADOS EN PANTALLA

### CSV desde la carpeta resources/data
<img width="1512" height="982" alt="image" src="https://github.com/user-attachments/assets/1deee4ef-139b-4693-a836-6e9038660454" />

### Resultado desde la terminal
<img width="1512" height="982" alt="image" src="https://github.com/user-attachments/assets/3310cb07-44ff-4e7b-9982-452516219fdf" />

### Resultado en MySql
<img width="1512" height="982" alt="image" src="https://github.com/user-attachments/assets/d9f00a76-418c-4b96-b891-5879d6dcd7d6" />




