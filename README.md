# CallMeMaybe — Identificación de operadores ineficaces

Proyecto final de análisis de datos: detección de operadores con desempeño potencialmente ineficaz en el servicio de telefonía virtual **CallMeMaybe**, a partir de EDA, ingeniería de métricas por operador y pruebas de hipótesis no paramétricas.

## Contenido del repositorio

```
├── proyecto_final_telecom_operadores_ejecutado.ipynb   # Notebook principal (EDA + análisis + pruebas)
├── telecom_dataset_new.csv                              # Registros de llamadas
├── telecom_clients.csv                               # Clientes y tarifa contratada
├── operator_inefficiency_metrics.csv                    # Métricas por operador (output)
├── presentacion.pdf                                      # Presentación ejecutiva de hallazgos
└── README.md
```

## Objetivo

Un operador se considera **ineficaz** si presenta:

- una tasa alta de llamadas entrantes perdidas, **combinada con**
- tiempos de espera prolongados en llamadas entrantes; **o**
- un volumen bajo de llamadas salientes, cuando su rol incluye hacer llamadas salientes.

El proyecto construye un criterio objetivo y reproducible para detectar este patrón y lo valida estadísticamente.

## Datos

**`telecom_dataset_us.csv`** — un registro por combinación cliente/fecha/dirección/operador:

| Columna | Descripción |
|---|---|
| `user_id` | ID de la cuenta cliente |
| `date` | Fecha de la estadística |
| `direction` | `in` (entrante) / `out` (saliente) |
| `internal` | Si la llamada fue entre operadores del mismo cliente |
| `operator_id` | ID del operador |
| `is_missed_call` | Si la llamada se perdió |
| `calls_count` | Número de llamadas |
| `call_duration` | Duración sin tiempo de espera |
| `total_call_duration` | Duración incluyendo espera |

**`telecom_clients_us.csv`** — `user_id`, `tariff_plan`, `date_start`.

Periodo cubierto: **2019-08-01 a 2019-11-27**, 307 clientes, 1,092 operadores identificados.

## Metodología

1. **Limpieza**: se eliminaron 4,900 filas duplicadas (53,902 → 49,002 registros). Los nulos en `internal` se marcaron como `"unknown"` en vez de descartarse; los nulos en `operator_id` (llamadas no asignadas a un operador, p. ej. cola/IVR) se excluyen solo del análisis por operador.
2. **EDA**: distribución de llamadas por dirección (75.5% salientes / 24.5% entrantes), por tipo (98.2% externas, 1.8% internas, 0.03% desconocidas), volumen diario y duración de llamada.
3. **Métricas por operador**: se agregan `calls` a nivel `user_id` + `operator_id`, calculando tasa de llamadas perdidas, espera promedio y volumen saliente por día.
4. **Filtro de actividad mínima**: solo se evalúan operadores con ≥10 llamadas totales y ≥3 días activos, para que operadores con poca actividad no distorsionen los percentiles (787 de 1,092 operadores calificaron).
5. **Criterio de ineficacia** (basado en percentiles de la propia población, no en umbrales arbitrarios):
   - Tasa de llamadas perdidas (entrante) ≥ percentil 75 (**1.66%**)
   - Espera promedio (entrante) ≥ percentil 75 (**20.6 s**)
   - Volumen saliente/día ≤ percentil 25 (**4.12**), solo si el operador tiene rol saliente inferido
   - Ineficaz = (tasa perdida alta **Y** espera alta) **O** bajo volumen saliente

## Resultados clave

- **194 de 787 operadores activos (24.6%)** se marcan como potencialmente ineficaces.
  - 41 por combinar tasa de perdidas alta + espera alta.
  - 163 por bajo volumen saliente.
- **Pruebas de hipótesis (Mann-Whitney U, α = 0.05)** — se usa esta prueba no paramétrica porque las distribuciones de llamadas y tiempos están sesgadas y con outliers:

| Hipótesis | Media grupo ineficaz | Media resto | p-value | ¿Rechaza H0? |
|---|---|---|---|---|
| H1: tasa diaria de llamadas perdidas es mayor | 2.35% | 0.88% | 0.00018 | Sí |
| H2: espera diaria promedio entrante es mayor | 21.87 s | 14.55 s | <0.0001 | Sí |
| H3: llamadas salientes diarias son menores | 3.04 | 49.85 | <0.0001 | Sí |

Las tres pruebas confirman que las diferencias observadas son estadísticamente significativas y no producto del azar.

## Recomendación de producto

Vista de supervisión con: ranking por `inefficiency_score`, filtros por cliente/tarifa/fecha/dirección, y motivo visible por operador (`high_missed_in`, `high_wait_in`, `low_outbound`). El resultado se plantea como una **señal de alerta para auditoría, revisión de turnos y capacitación**, sujeta a validación humana con los supervisores — no como criterio automático de decisión.

## Herramientas

Python · pandas · NumPy · Matplotlib · SciPy (`scipy.stats.mannwhitneyu`)

## Cómo ejecutar

```bash
pip install pandas numpy matplotlib scipy jupyter
jupyter notebook proyecto_final_telecom_operadores_ejecutado.ipynb
```

## Fuentes consultadas

1. [SciPy — `scipy.stats.mannwhitneyu`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.mannwhitneyu.html) — qué prueba usar sin supuesto de normalidad y cómo definir la hipótesis alterna (`greater`/`less`).
2. [Pandas — Merging guide](https://pandas.pydata.org/docs/user_guide/merging.html) — validar merges 1:N (`validate='many_to_one'`) sin duplicar filas.
3. [Matplotlib — `pie()` reference](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.pie.html) — evitar etiquetas encimadas en gráficos de pastel con categorías desbalanceadas.
4. [Verint — 5 Popular Call Center Benchmarks](https://www.verint.com/blog/5-popular-call-center-benchmarks-how-do-you-stack-up/) — referencia de industria para tasa de abandono aceptable.
5. [Geckoboard — Call Abandonment Rate](https://www.geckoboard.com/resources/kpi-examples/call-abandonment-rate/) — relación entre tiempos de espera largos y abandono de llamadas.
6. [LiveAgent — Call Center Industry Standard Metrics](https://www.liveagent.com/academy/best-practices-industry-standards/) — benchmark de industria (regla 80/20) usado como contraste.
7. *Storytelling with Data* — Cole Nussbaumer Knaflic — guía de estructura narrativa para la presentación ejecutiva.

## Presentación

La presentación ejecutiva con hallazgos, gráficas y recomendación está en [`presentacion.pdf`](https://drive.google.com/file/d/1CmWodMK1wTjtOhg_GXf4tJNfyN3PA53U/view?usp=sharing).

## Autor

Jesús David Palma García — CPA / Analista de datos
