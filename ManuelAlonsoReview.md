# Code Review

## ARQUITECTURA

La aplicación esta usando un intento de Arquitectura Hexagonal, pero se observan algunos errores, como por ejemplo que el dominio no está completamente aislado, por ejemplo, se están usando anotaciones Spring para inyectar dependencias, cuando el negocio nunca debería estar acoplado de esa manera.

Se ve como el AdsService está incluido en el paquete application cuando debería estar en un paquete domain o core, donde esté incluído toda la lógica de negocio.

Se ve como las clases AdVO y PictureVO, que son las clases que parece que se van a persistir, están en la infraestructura y no en el dominio.

## CLEAN CODE

Se observan algunos errores a la hora de realizar código lo más limpio y legible posible.

Personalmente quitaría los comentarios en mayor medida. Un código es limpio cuando se entiende por si solo la que está haciendo, siendo sus nombres de métodos y variables lo más descriptivo posible.

### CONSTANTES

El archivo Constants no es muy descriptivo. Ni en nombre ni en funcionalidad, ya que estos no dan mucha información sobre el objetivo de las mismas.

Si la idea es que sea el encargado de aplicar valores a sumar/restar a la hora de hacer cálculos para las puntaciones de anuncios, lo ideal sería cambiarlos.

Además, algunas puntuaciones se repiten, por ejemplo, una foto en HD da 20 puntos pero también puntua 20 puntos si la descripción es de más de 50 palabras y además es un chalet. Ejemplo:
  * HD_PICTURE_SCORE = 20.
  * CHALET_DESCRIPTION = 20.

Así nos aseguramos que en caso de tener que cambiar alguna puntuación, éstas no están tan acopladas entre sí.

También añadiría otros archivos de constantes. Por ejemplo, ahora mismo se otorgan 20 puntos si un chalet tiene más de 50 palabras en su descripción y quizás sea buena idea guardarlo en otro fichero constante.

También se podría valorar la idea de guardar estos valores en un archivo properties. Con eso conseguimos que si se cambia el valor de las puntuaciones, no haría falta volver a desplegar la aplicación en producción y podriamos cambiarlas en caliente.

### SOLID

El método AdsServiceImpl#calculateScore tiene una gran cantidad de condicionales que dificultan su comprensión y además quiebran el principio de abierto/cerrado al tener que tocar el método cada vez que se quiera cambiar o introducir un nuevo tipo de puntuación.

Propongo usar polimorfismo y el patrón Command para desacoplar el método respecto a los diferentes tipos de puntuaciones y conseguir devolver el principio de abierto/cerrado.

Ej:
```
@FunctionalInterface
public interface AdScorer {

    int score(Ad ad);

}
```
```
public class PictureScorerCommand implements AdScorer {
    @Override
    public int score(Ad ad) {
        return ad.getPictures().isEmpty()
                ? Constants.SCORE_NO_PICTURES : scoreByPicturesResolution(ad.getPictures());
    }

    private int scoreByPicturesResolution(List<Picture> pictures) {
        return pictures.stream()
                .mapToInt(value -> value.getQuality().equals(Quality.HD) ? Constants.SCORE_HD_PICTURE : Constants.SCORE_SD_PICTURE)
                .sum();
    }

}
```

Faltaría por crear otra clase que haga las funciones de Invoker para el resto de comandos, clase que sería llamada desde AdsServiceImpl#calculateScore

Se podría mejorar incluso más si añadimos un enumerado que relacione las diferentes calidades de fotos con los valores a puntuar. Mejorando incluso más el abierto/cerrado, ya que sólo tendriamos que consultar la calidad de foto en ese enumerado y obtener su valor, para así no tener que tocar el PictureScorerCommand a futuro.

Otro caso que se observa es una posible violación del principio de responsabilidad única, ya que AdsServiceImpl es el responsable también de mappear las entidades a los DTOs, cuando podría haber liberías mapper para ello o incluso estar esa lógica de mapeo en un constructor del DTO.

## TESTS

No se ha aplicado TDD en el proyecto puesto que está casi sin testear. Faltaría mucho para poder llegar a un porcentaje decente de código testado. 

Habría que añadir bastantes más además de añadir tests de integración, por ejemplo para la persistencia, usando H2 o para los controladores, usando WireMock y levantando el contexto de Spring.

## CORRECIONES Y MEJORAS EN EL CÓDIGO.
Se observan algunas cosas a corregir, entre ellas:
- Se está usando el verbo GET para el endPoint /score en AdsController#calculateScore, pero sin embargo, el flujo de este endPoint acaba en una actualización del anuncio en cuestión, por lo cual debería ser PATCH
- A la hora de añadir la puntuación en el anuncio, este sólo se hace con los datos que tengan en el momento de hacerla. Si a posteriori se cambiaran las imágenes o descipción, debería cambiar la puntuación, pero a no ser que se haga explicitamente, no lo hará. Quizás lo mejor sea refrescar la puntuación cada vez que se modifiquen las fotos o la descripción. Dejando 
- Añadir paginación en los listados de los anuncios
- Eliminar field injection por constructor injection o setter injection para no estar acoplados al contenedor IoC de Spring. Además facilita el uso de tests para esa clase, se publica las dependencias reales que tiene esta clase, se pueden crear esas inyecciones con el modificador final para hacerlas inmutables y se puede localizar más fácilmente si estamos rompiendo el principio de responsabilidad única al ver un constructor con demasiadas dependencias.
- En la AdsServiceImpl#100 se está seteando el valor en vez de sumarlo `score = Constants.FORTY;`
- A la hora de comprobar si ciertas palabras están en la descripción, se está comprobando siempre en minúsculas y con acentos además que hace un split por espacios, pero hay palabras que podrían estar junto a símbolos de puntuación y en ese caso, no contarían. Se podría mejorar e incluso meter ese listado en un application.properties o utilizarlo como un Map donde estarían la relación entre key (la palabra) y su value (la puntuación).
- Los métodos AdsServiceImpl#find
- Añadir una gestión de logs para registrar la aplicación.
- AdsServiceImpl#findPublicAds y AdsServiceImpl#findQualityAds podrían unificarse si mantenemos un enumerado que nos diga el valor que necesita cada tipo de anuncio para considerarse como tal, ya que luego sólo recoge de la "BBDD" los que superen ese filtro.
- AdVO y PictureVO me parece más entities que VOs al tener identidad propia. Además, habría que moverlas a mi parecer al dominio al ser objetos de dominio.
- Añadir una clase anotada con @RestControllerAdvice para el manejo de excepciones causadas desde una llamada a nuestra API REST y poder controlar mejor que dato queremos devolver.

## AÑADIDOS DE VALOR AL PROYECTO
Se podría añadir una serie de liberías y mejoras al proyecto para hacerlo más legible y eficiente:
  * Añadir alguna librería para mappear. Recomiendo MapStruct al ser annotation-based. Aunque hay bastantes otras para elegir como DozerMapper u OrikaMapper.
  * Añadir especificación para la API Rest. Se podría añadir OpenApi configurando un yaml o json. También se pueden añadir plugins a maven para que autogenere las interfaces de los controladores basándose en ese yaml.
  * Añadir Maven wrapper. Con esto conseguimos que se pueda lanzar las instrucciones maven sin importar si quien las ejecuta tiene la versión de maven correcta o siquiera si lo tiene.
  * Añadir la librería lombok para eliminar el boilerplate de los POJOs además de poder añadirles fácilmente a las clases que hagan falta el patrón Builder con una simple anotación.
  * Añadir versionado para la api. Se podría añadir en el application.properties y usarlo en los diferentes endPoints.

## ACTUALIZACIÓN JAVA
Aunque está en Java 8, en general, el código no usa las funcionalidades más modernas que ofrecen las últimas versiones java. Echo de menos más uso de Streams y de más optionals en caso de que algún campo pueda ser nulo. Por ejemplo, en el AdsServiceImpl#findQualityAds se podría utilizar
`adRepository.findIrrelevantAds().stream().map(QualityAd::new).collect(Collectors.toList())` Utilizando un constructor para QualityAd donde se le pasa un Ad o usar un mapeador. Con esto es más legible y además le quitamos la responsabilidad de mapear el QualityAd al servicio.

También se podría usar el nuevo tipo de clase record para crear inmutabilidad y claridad al eliminar boilerplate en algunas de las clases como en PublicAd o QualityAd.

Además, se está usando la antigua librería Date la cuál está deprecada. Habría que cambiarla por LocalDateTime que además es thread-safe.


