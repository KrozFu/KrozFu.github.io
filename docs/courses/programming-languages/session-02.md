# Serialización y Objetos Remotos

## Por qué un objeto no viaja tal cual

- Un objeto en memoria es una **estructura de punteros**: la dirección de un `dict`, que apunta a cadenas, que apuntan a bytes. Esas direcciones solo significan algo **dentro de este proceso**.
- Para enviarlo hay que **aplanarlo en bytes** _(serialización o *marshalling*)_ y reconstruirlo en el otro extremo *(deserialización)*.
- **Dos preguntas que ordenan el bloque:**
  1. ¿Quién puede leer los bytes? *(¿solo Python, o cualquiera?)*
  2. ¿**De quién** vienen los bytes? *(¿puedo confiar en ellos?)*
