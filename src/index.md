---
toc: false

sql:
  delitos: data/combined_data.parquet
  poblacion: data/comunasPoblacion.tsv
  maxValues: data/maxValues.tsv
---

# Tasa de Delitos por comuna en Chile
Esta página muestra la tasa de delitos (cada 100 mil habitantes) para la comuna seleccionada a lo largo del tiempo.  

La cifra de delitos se refiere a la tasa de **Casos Policiales** en los últimos 12 meses (se suma la tasa de los últimos 4 trimestres para cada punto en el tiempo asociado a un trimestre). 

**Casos Policiales** "Es el indicador utilizado para analizar la ocurrencia de hechos delictivos. Considera las denuncias de delitos que realiza la comunidad en las unidades policiales, más las detenciones que realizan las policías ante la ocurrencia de delitos flagrantes. Internacionalmente este indicador es conocido como 'delitos conocidos por la policía' (crimes known to police). Esta información está disponible desde el año 2005.", CEAD



```js
const familiaDelitoSeleccionada = view(Inputs.select(_.keys(tiposDelitos), {
  label: "Familia Delito"
}));
```


```js
const grupoDelitoSeleccionado = view(Inputs.select(
  [`Todos (${familiaDelitoSeleccionada})`].concat(
    tiposDelitos[familiaDelitoSeleccionada]
  ),
  {
    label: "Grupo Delito"
  }
));

```

```js
const delitoSeleccionado = grupoDelitoSeleccionado.match(/^Todos/)
  ? familiaDelitoSeleccionada
  : grupoDelitoSeleccionado
```

```js
const comunaSeleccionada = view(Inputs.select(  
  _.chain(comunas)
    .value(),
  {
    label: "Comuna",
    format: (d) => `${d.Comuna} (${d3.format(".3s")(d.población)} habs.)`
  }));
```
*Nota: se incluyen comunas con más de 100 mil habitantes, y en el gráfico se agregan comunas re referencia según las cifras en los últimos 6 trimestres*

<div class="card">
<div>${chart}</div>
<div style="color:grey;">Fuente de datos: CEAD, Centro de Estudios y Análisis del Delito, https://cead.spd.gov.cl/estadisticas-delictuales</div>
</div><!--card-->

```js
const chart = (() => {
  const comunaFoco = comunaSeleccionada["Comuna"];
  const opacityReference = 0.4;
  const dataPlot = dataReferencia
    .map((d) => {
      const record = {
        Comuna: d.Comuna,
        Delito: d.Delito,
        Valor: d.Valor,
        fecha: convertToChartDate(d.Año, d.Trimestre),
        Año: d.Año,
        Trimestre: d.Trimestre
      };
      return record;
    })
    .filter((d) => d.Delito.match(/|/))
    .filter((d) => d.fecha <= convertToChartDate("2024", "TRIM 2"))


  const rollsum = Plot.window({
    k: 4,
    anchor: "end",
    reduce: "sum",
    strict: true
  });

  const maxYValue = 
    delitoSeleccionado.match(/Violaciones y delitos sexuales/) && comunaSeleccionada.Comuna.match(/Alto Hospicio/) 
    ? 800 
    : delitoSeleccionado.match(/simples delitos ley de armas/) && comunaSeleccionada.Comuna.match(/Quinta Normal/)
    ? 1100
    : delitoSeleccionado.match(/Delitos asociados a armas/) && comunaSeleccionada.Comuna.match(/Quinta Normal/)
    ? 1100
    : maxValue
   
  return Plot.plot({
    title: `${delitoSeleccionado == familiaDelitoSeleccionada ? delitoSeleccionado : `${familiaDelitoSeleccionada} - ${delitoSeleccionado}`}`,
    subtitle: `${comunaSeleccionada.Comuna}`,
    marginRight: 100,

    x: { grid: true },
    y: {
      grid: true,
      domain: [0, maxYValue],
      label: "Delitos cada 100 mil habitantes (últimos 12 meses)"
    },
    color: { legend: true, domain: [comunaFoco].concat(comunasReferencia) },
    width,
    marks: [
      Plot.ruleY(
        dataPlot.filter((d) => d.Comuna == comunaSeleccionada.Comuna),
        Plot.selectLast(
          Plot.windowY(
            { k: 4, reduce: "sum", anchor: "end" },
            {
              y: "Valor",
              stroke: (d) => d.Comuna,
              strokeDasharray: "1",
              sort: (d) => d["fecha"].toDate()
            }
          )
        )
      ),
      Plot.ruleY([0]),
      Plot.lineY(
        dataPlot,
        Plot.windowY(
          { k: 4, anchor: "end", reduce: "sum", strict: true },
          {
            x: (d) => d["fecha"].toDate(),
            y: "Valor",
            stroke: "Comuna",
            strokeWidth: (d) => (d.Comuna == comunaFoco ? 2 : 1),
            strokeOpacity: (d) =>
              d.Comuna == comunaFoco ? 1 : opacityReference,
            sort: (d) => d["fecha"].toDate()
          }
        )
      ),
      Plot.dot(
        dataPlot,
        Plot.map({
            title: (index, series) => {
              /*eturn index.map(
                (i) => `${dataPlot[i].Comuna}\n${d3.format(".1f")(series[i])} delitos x 100 mil habs.`
              );*/
              return index.map(
                (i) => `${dataPlot[i].Comuna}\n${mapaTrimestreMes[dataPlot[i].Trimestre]} ${dataPlot[i].Año}\n${d3.format(".1f")(series[i])} delitos x 100 mil habs. en 12 meses`
              );
            }
          },
        Plot.map(
          { y:rollsum, title:rollsum },
          {
            x: (d) => d["fecha"].toDate(),
            y: "Valor",
            fill: "Comuna",
            fillOpacity: (d) => (d.Comuna == comunaFoco ? 1 : opacityReference),

            r: 1.2,
            tip: true,
            title: "Valor"
          }
        )
        )
      ),
      Plot.text(
        dataPlot,
        Plot.selectLast(
          Plot.windowY(
            { k: 4, anchor: "end", reduce: "sum" },
            {
              x: (d) => d["fecha"].toDate(),
              y: "Valor",
              z: "Comuna",
              text: "Comuna",
              textAnchor: "start",
              fillOpacity: (d) =>
                d.Comuna == comunaFoco ? 1 : opacityReference,

              dx: 5,
              sort: (d) => d["fecha"].toDate()
            }
          )
        )
      )
    ]
  });
})()



```

### Comunas de Referencia (Julio 2023 a Junio 2024)
*Se consideran 63 comunas con más de 100 mil habitantes*
```js
Inputs.table(quartiles)
```


```js

const comunasReferencia2 =  [
  "Santiago",
  "Las Condes",
  "La Florida",
  "La Pintana",
  "Punta Arenas"
]

const comunasList = comunasReferencia
  .concat(comunaSeleccionada["Comuna"])
  .map((d) => `'${d}'`)
  .join(",");

const mapaTrimestreMes = {
 "TRIM 1": "Marzo",
 "TRIM 2": "Junio",
 "TRIM 3": "Septiembre",
 "TRIM 4": "Diciembre",
}

```

```js

const tiposDelitos = {
  "Delitos violentos": [
    "Homicidios y femicidios",
    "Violaciones y delitos sexuales",
    "Robos con violencia o intimidación",
    "Robo por sorpresa",
    "Lesiones graves o gravísimas",
    "Lesiones menos graves",
    "Lesiones leves",
    "Violencia intrafamiliar",
    "Amenazas con armas",
    "Amenazas o riña"
  ],
  "Delitos asociados a drogas": [],
  "Delitos asociados a armas": [
    "Crímenes y simples delitos ley de armas",
    "Porte de arma cortante o punzante"
  ],
  "Delitos contra la propiedad no violentos": [
    "Robos en lugares habitados y no habitados",
    "Robos de vehículos y sus accesorios",
    "Otros robos con fuerza en las cosas",
    "Hurtos",
    "Receptación"
  ],
  Incivilidades: [
    "Consumo de alcohol y drogas en la vía pública",
    "Daños",
    "Desórdenes públicos",
    "Otras incivilidades"
  ],
  "Otros delitos o faltas": []
}
```

```js
const result = await sql([`
  SELECT * 
  FROM delitos 
  WHERE Comuna IN  (${comunasList}) and Delito = '${delitoSeleccionado}'
  ORDER BY Año, Trimestre
  `]);
const dataReferencia = [...result]

```


```sql id=[{maxValue}] 
SELECT MaxRollingSum as maxValue
FROM maxValues 
WHERE Delito = ${delitoSeleccionado}
LIMIT 1
```




```sql id=delitos 
SELECT DISTINCT Delito
FROM delitos
```


```sql id=comunas 
SELECT Comuna, población
FROM poblacion
WHERE población > 100000
ORDER BY Comuna
```


```js

const quartiles = await sql([`
WITH datos as (SELECT delitos.Comuna, poblacion.población, SUM(Valor) as Valor
FROM delitos
INNER JOIN poblacion ON delitos.Comuna = poblacion.Comuna
  WHERE población > 100000 AND Delito = '${delitoSeleccionado}' AND ((Año = 2023 AND Trimestre IN ('TRIM 3', 'TRIM 4')) OR Año = 2024)
GROUP BY delitos.Comuna, población
ORDER BY Valor),



quartiles AS (
    SELECT
        MIN(Valor) AS MinValue,
        PERCENTILE_DISC(0.25) WITHIN GROUP (ORDER BY Valor) AS Q1,
        PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY Valor) AS Median,
        PERCENTILE_DISC(0.75) WITHIN GROUP (ORDER BY Valor) AS Q3,
        MAX(Valor) AS MaxValue
    FROM datos
)

SELECT 
    Comuna,
    Valor as "Tasa de delitos",
    CASE
        WHEN ROUND(Valor,4) = ROUND(q.MinValue,4) THEN 'Min'
        WHEN ROUND(Valor,4) = ROUND(q.Q1,4) THEN 'P25'
        WHEN ROUND(Valor,4) = ROUND(q.Median,4) THEN 'P50 (Mediana)'
        WHEN ROUND(Valor,4) = ROUND(q.Q3,4) THEN 'P75'
        WHEN ROUND(Valor,4) = ROUND(q.MaxValue,4) THEN 'Max'
    END AS "Posición"
FROM 
    datos AS t
CROSS JOIN 
    quartiles AS q
WHERE 
    ROUND(Valor,4) = ROUND(q.MinValue,4) OR 
    ROUND(Valor,4) = ROUND(q.Q1,4) OR   
    ROUND(Valor,4) = ROUND(q.Median,4) OR   
    ROUND(Valor,4) = ROUND(q.Q3,4) OR   
    ROUND(Valor,4) = ROUND(q.MaxValue,4)
ORDER BY 
    Valor;
`])

const comunasReferencia = [...quartiles].map(d => d.Comuna)
```




```js
function convertToChartDate(year, trimester) {
  // Map each trimester to the starting month
  const trimesterStartMonths = {
    "TRIM 1": 3, // January
    "TRIM 2": 6, // April
    "TRIM 3": 9, // July
    "TRIM 4": 12 // October
  };

  // Get the starting month for the given trimester
  const month = trimesterStartMonths[trimester];

  // Use moment to create a date at the start of the trimester
  return moment([month == 12 ? year + 1 : year, month == 12 ? 0 : month, 1]); // Returns a Moment object
}

import moment from 'npm:moment'
```

