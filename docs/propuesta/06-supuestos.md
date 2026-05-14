# 6. Supuestos

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

1. **ANPR existente.** El componente de ANPR del dispositivo expone su salida por al menos uno de: REST local, socket UNIX, archivo plano vigilable o named pipe. El agente Edge implementa un *adapter SPI* para cada modalidad. Si no fuera así, se requiere una conversación con el fabricante para acordar un contrato de salida.

2. **APN privado o IP enrutable.** Los dispositivos pueden alcanzar los endpoints públicos del sistema (MQTT broker y object storage) directamente sobre GSM, idealmente vía APN privado del operador. En caso contrario, se interpone un edge proxy regional.

3. **Volumen de captura.** Se asume promedio de ~1 000 placas/día por dispositivo, con picos de hasta 10 000 en intersecciones de alto flujo. Esto se valida en pruebas piloto antes del rollout.

4. **Imágenes.** Las cámaras producen imágenes JPEG razonablemente comprimidas (≤ 500 KB full, ≤ 50 KB thumbnail). En caso contrario, el agente aplica compresión adicional.

5. **Reloj del dispositivo.** Existe un RTC que sobrevive cortes de energía; cuando hay conectividad se sincroniza vía NTP. Los eventos llevan timestamp wall-clock y monotonic para forensía.

6. **Disponibilidad GSM.** Hay cobertura suficiente para sincronizar varias veces al día en promedio. Casos extremos (zonas sin cobertura por semanas) requieren cosecha física, fuera de alcance de esta propuesta.

7. **Compromiso institucional.** Cada policía proporciona:
   - Acceso programático a su BD de hurtos (cuenta de servicio, credencial, ventana de mantenimiento conocida).
   - Endpoints o credenciales para federación de autenticación.
   - Persona técnica de contraparte para onboarding del adapter.

8. **Modelo canónico extensible.** Los nueve campos del enunciado (matrícula, fecha, lugar, DNI propietario, marca, clase, línea, color, modelo) son obligatorios. Los campos extra de cada país viajan en `extensions: jsonb` y se documentan por país.

9. **Imágenes de evidencia.** Se almacena la imagen completa más una *thumbnail* indexada. No se almacenan rostros ni se hace reconocimiento facial — fuera de alcance y de cumplimiento.

10. **Retención por defecto.** Eventos crudos 24 meses; imágenes full 6 meses; thumbnails 24 meses; audit log 7 años. Configurable por país.

11. **Operador del sistema.** Ceiba (o quien opere) cuenta con un equipo SRE / DevOps capaz de operar Kubernetes y servicios distribuidos. De no haberlo, se ajusta el plan operativo y se evalúan servicios gestionados con la abstracción descrita.

12. **Provisioning de dispositivos.** El fabricante incluye un bootstrap token de un solo uso por unidad (o un certificado de fábrica) que permite intercambiarlo por el certificado operativo emitido por Vault PKI.

13. **OTA.** Existe ventana para enviar actualizaciones del agente; los dispositivos toleran reinicios controlados.

14. **Idioma.** Las UIs son al menos español/inglés. La internacionalización está prevista en el frontend desde el día uno.

15. **Tiempo a producción.** Se asume un piloto inicial con 1–2 países y 1 000–5 000 dispositivos antes del rollout regional. La propuesta no implica que todo lo descrito se construya en la primera iteración: muchas piezas (analítica avanzada, multi-región DR, federaciones SOAP) llegan en fases posteriores.

16. **Modo de captura del agente (`upload_mode`).** Por defecto el agente sube al cloud **únicamente los eventos con hit en el Bloom filter** (`upload_mode: stolen_only`). Esto minimiza el volumen de datos, el consumo de banda GSM y la exposición de datos de tránsito. El parámetro es **configurable remotamente** vía Config/OTA Manager; con `upload_mode: all` el agente sube todos los eventos. El cambio de modo requiere autorización institucional explícita del país operador, dado que `all` implica registro masivo de tránsito vehicular.
