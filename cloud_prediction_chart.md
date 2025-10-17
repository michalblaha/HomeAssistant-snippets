# Cloud cover prediction chart with highlighted daytime hours

Cloud cover prediction chart with highlighted periods when the sun is shining on the PV plant (3 hours after sunrise, 1 hour before sunset)

The chart shows the cloud cover forecast from met.no and OpenWeatherMap.

```yaml
type: custom:apexcharts-card
grid_options:
  columns: full
  rows: auto
header:
  show: true
  title: Oblačnost – další 2 dny
graph_span: 48h
span:
  end: hour
  offset: +48h
now:
  show: true
  label: Teď
yaxis:
  - min: 0
    max: 100
    decimals: 0
series:
  - entity: sensor.weather_forecast_home_cache_met_no
    name: Denní okno
    type: area
    color: "#9e9e9e"
    opacity: 0.25
    stroke_width: 0
    data_generator: |
      // Načti časy východu/západu (ber jen čas, datum ignoruj)
      const riseStr = hass?.states?.['sensor.sun_next_rising']?.state ?? null;
      const setStr  = hass?.states?.['sensor.sun_next_setting']?.state ?? null;

      // Fallback: když chybí senzory, nic nezvýrazňuj
      if (!riseStr || !setStr) return [];

      const rise = new Date(riseStr);
      const set  = new Date(setStr);

      // Hodiny/minuty pro okno: sunrise +3h, sunset -1h
      let startH = rise.getHours() + 3;
      let startM = rise.getMinutes();
      let endH   = set.getHours() - 1;
      let endM   = set.getMinutes();

      // Korekce přetečení/underflow
      if (startH >= 24) { startH -= 24; }
      if (endH   < 0)   { endH   += 24; }

      const arr = entity?.attributes?.forecast ?? [];

      return arr
        .filter(x => x?.datetime)
        .map(x => {
          const t = Date.parse(x.datetime);
          const d = new Date(t);

          // Sestav okno pro konkrétní DEN timestampu t (ignoruj datum ze senzorů)
          const start = new Date(d);
          start.setHours(startH, startM, 0, 0);

          const end = new Date(d);
          end.setHours(endH, endM, 0, 0);

          // Pokud by okno překročilo půlnoc (teoreticky), ošetři:
          let inWindow;
          if (end <= start) {
            // okno start..24:00 nebo 0:00..end
            inWindow = (d >= start) || (d <= end);
          } else {
            inWindow = (d >= start) && (d <= end);
          }

          // Vyplň 100 (plná plocha) jen uvnitř okna, jinak pauza
          return [t, inWindow ? 100 : null];
        });
  - entity: sensor.weather_forecast_home_cache_met_no
    name: Met.No %
    type: line
    stroke_width: 2
    data_generator: |
      const arr = entity?.attributes?.forecast ?? [];
      return arr
        .filter(x => x?.datetime && x?.cloud_coverage != null)
        .map(x => [Date.parse(x.datetime), Number(x.cloud_coverage)])
        .filter(([t,y]) => Number.isFinite(t) && Number.isFinite(y));
  - entity: sensor.weather_forecast_home_cache_openweathermap
    name: OpenWeatherMap %
    type: line
    stroke_width: 2
    data_generator: |
      const arr = entity?.attributes?.forecast ?? [];
      return arr
        .filter(x => x?.datetime && x?.cloud_coverage != null)
        .map(x => [Date.parse(x.datetime), Number(x.cloud_coverage)])
        .filter(([t,y]) => Number.isFinite(t) && Number.isFinite(y));
apex_config:
  chart:
    height: 250px
  xaxis:
    type: datetime
    min: new Date().getTime()
    max: new Date(new Date().getTime() + 48*3600*1000).getTime()
    labels:
      datetimeUTC: false
  markers:
    size: 2
  annotations:
    position: back
    xaxis:
      - x: new Date().getTime()
        x2: new Date(new Date().setHours(24,0,0,0)).getTime()
        fillColor: rgba(0, 123, 255, 0.8)
        opacity: 0.95
      - x: new Date(new Date().setHours(24,0,0,0)).getTime()
        x2: new Date(new Date().setHours(48,0,0,0)).getTime()
        fillColor: rgba(40, 167, 69, 0.8)
        opacity: 0.95

```
