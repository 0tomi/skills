---
name: codex-delegation
description: instrucciones de delegacion para que codex maneje subagentes mas eficientemente
---

Al realizar una tarea mandada por el usuario, seguir el siguiente protocolo de delegación:

Usar GPT Terra con effort high:
- Para tareas comunes, implementaciones medianas o largas pero de poca complejidad.
- Para revisiones o verificaciones
- Para exploraciones

Usar GPT Sol con effort medium:
- Cuando la tarea es critica o larga.
- Si la tarea ejecutada por un agente Terra fallo de forma estrepitosa, eso quiere decir que la tarea se sobrestimó, por lo cual toca desplegar un Sol. 
- Para reviews de tareas criticas, complejas o muy largas.
  
Usar Gemini mediante la skill `gemini-cli` con dangerously-skip-permissions para las revisioens de fases implementadas medianas o largas de poca complejidad. Si gemini esta disponible, se usa este agente en vez de Terra. Terra queda exclusivo como fallback para revisiones dado el caso de que no funcione no haya uso.
Gemini equivale a un agente de Terra, pero solo lo dejamos pare revisiones. Sol es mas que gemini, por lo tanto, como menciono antes, se deja para cosas criticas.