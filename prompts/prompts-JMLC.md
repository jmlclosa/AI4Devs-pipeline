# Prompts usando Jetbrains AI Assistant en IntelliJ

## Prompt para pedir una acción con los pasos checkout + test + build

```
Actúa como un devops experto en Github actions.
Necesito implementar una Github action en el fichero #file:ci.yml  que se ejecute cuando se haga un push en una rama con un Pull Request abierto

Los pasos que se deben seguir son:
1. Checkout
3. Ejecutar tests del módulo backend
4. Generar una build del módulo backend

Puedes utilizar el fichero #file:README.md  para saber como se compila y lanza el proyecto.

Sigue buenas prácticas en todo momento de seguridad, legibilidad, etc.
```