# Post-contenido — Unidad 7: Patrones Arquitectónicos I

## Descripción
Repositorio del post-contenido de la Unidad 7 de Patrones de Diseño de Software. Un único proyecto Spring Boot (`multas-biblioteca-api`) para la gestión de multas de biblioteca, con dos partes: una API REST en capas (Model, Repository, Service, Controller) sobre H2, y pago en línea de multas con dos pasarelas intercambiables.

## Parte 1 — Arquitectura en Capas
`MultaRepository` extiende `JpaRepository` y agrega una consulta agregada (`countByEstudianteIdAndEstado`). `MultaService` concentra las reglas de negocio (límite de multas pendientes, generación); el cálculo del monto vive en la propia entidad `Multa` (`Multa.calcularMonto`). `MultaController` expone `/api/multas`. Ver paquetes `model/`, `repository/`, `service/` y `controller/`.

## Parte 2 — Pago en Línea con Dos Pasarelas
Se implementó la **Opción C: Puerto de dominio con dos adaptadores (Hexagonal / Puertos y Adaptadores)**. Se definieron `domain/port/PasarelaPagoPort` y `domain/ResultadoPago` en Java puro, sin ninguna dependencia de Spring Framework. En la capa `infrastructure/pago/`, `PagosUdesAdapter` y `WompiAdapter` traducen sus respectivos contratos HTTP al tipo de dominio unificado `ResultadoPago`. El adaptador activo se selecciona automáticamente al arrancar mediante la propiedad `app.pagos.proveedor` (`@ConditionalOnProperty`), permitiendo cambiar de pasarela sin necesidad de modificar ni re-compilar `MultaController`, `MultaService` ni el resto de la aplicación.

## Cómo ejecutar
```bash
$ cd multas-biblioteca-api && ./mvnw spring-boot:run
```

# Herramientas utilizadas
Java 17, Spring Boot 3.x, Spring Data JPA, H2, RestTemplate

Apache Maven, Postman/cURL, Git, GitHub

# Decisiones de diseño
## Punto de decisión 1 — Cálculo del monto: ¿entidad o Service?
El cálculo del monto vive en el método estático de fábrica de la entidad `Multa` (`Multa.calcularMonto`) y no en `MultaService` para evitar caer en el antipatrón de un Modelo Anémico. La tarifa diaria ($500) y el tope máximo ($15,000) son reglas intrínsecas a la definición de una multa; por tanto, la entidad debe encapsular tanto sus datos como la lógica necesaria para garatizar que nace en un estado coherente. Si esta lógica estuviera en el servicio, la entidad pasaría a ser un simple contenedor de datos sin comportamiento.

## Punto de decisión 2 — Conteo de multas pendientes: ¿consulta o filtrado en memoria?
Se optó por delegar la regla a la base de datos mediante el método derivado `countByEstudianteIdAndEstado` en `MultaRepository`. Filtrar las multas trayéndolas en memoria hacia la aplicación requeriría cargar todas las entidades del estudiante desde H2 antes de contarlas, lo cual introduce un desperdicio ineficiente de memoria y procesamiento. La consulta agregada realiza la operación `COUNT(*)` directamente en el motor de base de datos de manera atómica y eficiente.

## Punto de decisión 3 — Selección del adaptador activo
Se utilizó `@ConditionalOnProperty` para resolver la selección del adaptador en tiempo de arranque. Dado que el requisito especifica una pasarela fija por sede durante la fase piloto, solo un Bean de tipo `PasarelaPagoPort` (`PagosUdesAdapter` o `WompiAdapter`) es registrado en el contexto de Spring. La alternativa —inyectar un `Map<String, PasarelaPagoPort>` y seleccionar la pasarela dinámicamente en tiempo de ejecución— requería que `MultaService` conociera explícitamente las claves de configuración de cada proveedor (p. ej. "pagosudes" o "wompi"), acoplando el servicio a los detalles de infraestructura.

## Punto de decisión 4 — Diseño del puerto y el tipo de resultado
El record `ResultadoPago` es agnóstico a las pasarelas y expone únicamente atributos neutros del dominio (proveedor, exitoso, referenciaExterna, mensaje). `PagosUDES` devuelve `idTransaccion` y `Wompi` trabaja con reference y montos en centavos. Si `ResultadoPago` hubiera expuesto un atributo específico de una pasarela (como `idTransaccion` en lugar de `referenciaExterna`), `WompiAdapter` habría tenido que mapear forzadamente su respuesta a un nombre ajeno a su contrato o dejarlo incoherente. El diseño neutro evita que la interfaz del puerto sufra modificaciones al añadir una tercera pasarela.

# Trade-off considerado — Parte 2
Para la Parte 2 se consideró y descartó la Opción B (Interfaz Strategy en la capa Service) a favor de la Opción C (Puertos y Adaptadores).

**Lo que se ganó**: Un aislamiento total entre la lógica de negocio y las librerías HTTP/Spring, permitiendo probar `MultaService` en aislamiento completo y garantizar que los contratos heterogéneos de cada pasarela no contaminen el dominio.

**El costo adicional**: Se introdujo una mayor cantidad de clases y paquetes (`domain/` e `infrastructure/`), aumentando la verbosidad inicial del proyecto y requiriendo un mejor entendimiento de la inversión de dependencias.

**Evaluación post-piloto**: Si el piloto concluye y la universidad decide quedarse con una única pasarela definitiva (p. ej. PagosUDES), el equipo no revertiría la decisión. El puerto `PasarelaPagoPort` mantendría la arquitectura desacoplada y lista para futuros cambios sin representar una sobrecarga mantenible significativa.

## Conclusiones
La implementación combinada de una arquitectura en capas tradicional con el patrón de Puertos y Adaptadores permitió resolver de forma limpia dos necesidades de naturaleza distinta en un mismo sistema. La mayor dificultad al elegir la arquitectura de la Parte 2 consistió en balancear la simplicidad del código contra la extensibilidad a futuro, determinando si la diferencia de contratos entre PagosUDES y Wompi justificaba crear un paquete de dominio puro. Finalmente, la separación clara de responsabilidades demostró que Inversión de Dependencias no solo protege el core de negocio contra cambios externos, sino que facilita las pruebas unitarias y el mantenimiento del software a largo plazo.